# Massive Ad Client [DRAFT]

23/08/25 - very early version

---

## INTRO

## HISTORY

## Known Versions

At this point in time, thanks to website "dlldownloader" and a lot of games I was able to get in my posession 4 out of 5 known versions of the MAD:

- `m4d.dll` with version 1.2 of MAD
- `MassiveAdClient.dll` with version 2.4
- Need For Speed: Carbon 1.4 with versions 3.2
- Sk8te 2 for Xbox 360 (with debug symbols!) with version >3.2

I've zero idea where first 2 dlls are coming from, and I'm not sure any interesting games utilizing it. So I focused on version 3.5, while utilizing Sk8te 2 symbols to aid me in reversing stuff. Plus from version 4 they switched to https, with ciphers which are now not supported by pretty much anything modern. So I decided to tackle version 3 first, and then work with version 4.

## First packet

On load, NFSC sends a http request to `http://locate.madserver.net/advsrc/locateService`.

If we check the content of data being sent:

```
POST /adsrv/locateService HTTP/1.1
Host:127.0.0.1
Content-Length:31
Content-Type:application/massive
User-Agent:Adclient Massive Inc./3.2.1.55

É=nfs_carbon_pc_na>0.0
```

This payload looks a little bit off. Let's check raw bytes.

```
03c9000000193d0010
6e66735f636172626f6e5f70635f6e61 - nfs_carbon_pc_na
3e0003
302e30 - 0.0
```

Okay. Now things are getting interesting. This is a raw bytes buffer! Curious approach by Massive.

After some snooping around in Sk8te 2 I was able to track down the function responsible for constructing said buffer:

```C++
void WriteLocateServiceRequest@CRequestLocateService@MassiveAdClient3
               (int param_1,undefined8 param_2,undefined8 param_3)

{
  undefined8 uVar1;
  size_t sVar2;
  char *local_30 [12];
  
                    /* Writing Request... */
  ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
            (5,*(undefined4 *)(param_1 + 0xc),0xffffffff8236faec);
  ?WriteU8@CRequestObject@MassiveAdClient3@@IAAXEH@Z(param_1,0x3d,0);
  ?WriteString@CRequestObject@MassiveAdClient3@@IAAXPADH@Z(param_1,param_2,0);
                    /* Writing SKU Name: %s */
  ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
            (5,*(undefined4 *)(param_1 + 0xc),0xffffffff8236fbdc,param_2);
  ?WriteU8@CRequestObject@MassiveAdClient3@@IAAXEH@Z(param_1,0x3e,0);
  ?WriteString@CRequestObject@MassiveAdClient3@@IAAXPADH@Z(param_1,param_3,0);
                    /* Writing SKU Version: %s */
  ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
            (5,*(undefined4 *)(param_1 + 0xc),0xffffffff8236fbc4,param_3);
  local_30[0] = (char *)0x0;
  uVar1 = ?Instance@CMassiveSystem@MassiveAdClient3@@SAPAV12@XZ();
  ?GetHardwareAddress@CMassiveSystem@MassiveAdClient3@@QAAXPAPAD@Z(uVar1,local_30);
  if (local_30[0] != (char *)0x0) {
    sVar2 = strlen(local_30[0]);
    if (sVar2 != 0) {
      uVar1 = CalculateMD5Hash(local_30[0]);
      ?WriteU8@CRequestObject@MassiveAdClient3@@IAAXEH@Z(param_1,0x1d,0);
      ?WriteString@CRequestObject@MassiveAdClient3@@IAAXPADH@Z(param_1,uVar1,0);
                    /* Writing Hashed Hardware ID: %s */
      ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
                (5,*(undefined4 *)(param_1 + 0xc),0xffffffff8236fba4,uVar1);
      (*(code *)PTR_free_84494774)(local_30[0]);
      goto LAB_834b1648;
    }
  }
                    /* Could not get Hardware ID from system. */
  ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
            (2,*(undefined4 *)(param_1 + 0xc),0xffffffff8236fb7c);
LAB_834b1648:
  ?FinishBaseBlock@CRequestObject@MassiveAdClient3@@IAAXEHH@Z(param_1,0xc9,0,0);
  return;
}
```

And things become crystal clear! Thanks symbols, you're trully a biggest gift to humanity.

Using the function above we're able to fully understand the payload:

```
03 - unknown
c9000000 - task id (Locate service)
19 - payload size
3d00 - field id (SKU name)
10 - string size
6e66735f636172626f6e5f70635f6e61 - SKU name (nfs_carbon_pc_na)
3e00 - field id (SKU version)
03 - string size
302e30 - SKU version (0.0)
```

Great, now we can implement some sort of a parsing. For server I decided to use my beloved `actix-web` web server framework and bytes reader via `scroll`.

```rust
async fn locate_service_v3(
    mut req: HttpRequest,
    mut body: web::Payload,
) -> actix_web::Result<String> {
    let mut bytes: BytesMut = web::BytesMut::new();
    while let Some(item) = body.next().await {
        bytes.extend_from_slice(&item?);
    }
    let hex_string = hex::encode(bytes.clone());
    info!("payload: {hex_string}");

    let mut bytes_cur = Cursor::new(bytes.clone());

    let unk1: u8 = bytes_cur.ioread::<u8>().unwrap();
    let task_id: u32 = bytes_cur.ioread::<u32>().unwrap();

    let payload_size: u8 = bytes_cur.ioread::<u8>().unwrap();

    let sku_name_field: u16 = bytes_cur.ioread::<u16>().unwrap();

    let sku_name_size: u8 = bytes_cur.ioread::<u8>().unwrap();

    let mut sku_name_vec: Vec<u8> = Vec::with_capacity(sku_name_size.into());

    for i in 0..sku_name_size {
        let mut buf: [u8; 1] = [0; 1];
        bytes_cur.read(&mut buf);
        sku_name_vec.push(buf[0]);
    }

    let sku_name: String = String::from_utf8(sku_name_vec).unwrap();

    let sku_version_field: u16 = bytes_cur.ioread::<u16>().unwrap();

    let sku_version_size: u8 = bytes_cur.ioread::<u8>().unwrap();
    let mut sku_version_vec: Vec<u8> = Vec::with_capacity(sku_version_size.into());

    for i in 0..sku_version_size {
        let mut buf: [u8; 1] = [0; 1];
        bytes_cur.read(&mut buf);
        sku_version_vec.push(buf[0]);
    }

    let sku_version: String = String::from_utf8(sku_version_vec).unwrap();

    info!("Task id: {task_id} | payload_size: {payload_size} | sku_name: {sku_name} | sku_version: {sku_version}");

    Ok(format!("Request Body Bytes:\n{:?}", bytes))
}
```

Quick and dirty, but it gets the job done.

```
[2025-06-06T06:11:44Z INFO  mad_server] payload: 03c9000000193d00106e66735f636172626f6e5f70635f6e613e0003302e30
[2025-06-06T06:11:44Z INFO  mad_server] Task id: 201 | payload_size: 25 | sku_name: nfs_carbon_pc_na | sku_version: 0.0
[2025-06-06T06:11:44Z INFO  actix_web::middleware::logger] 127.0.0.1 "POST /adsrv/locateService HTTP/1.1" 200 74 "-" "Adclient Massive Inc./3.2.1.55" 0.001149
```

Now it's time to figure out the reply for this request.

## Reply to LocateService request

Just below `WriteLocateServiceRequest` I found `Parse@CRequestLocateService@MassiveAdClient3` and it's a rather long function.

<details>
  <summary>.Game::C_GameWorld::Update</summary>

```C++
void Parse@CRequestLocateService@MassiveAdClient3(int param_1)

{
  uint uVar1;
  int iVar5;
  char cVar10;
  ulonglong uVar2;
  byte bVar11;
  ushort uVar8;
  longlong lVar3;
  void *pvVar6;
  undefined4 *puVar7;
  undefined2 uVar9;
  undefined8 uVar4;
  char local_a0;
  
  local_a0 = '\0';
  *(undefined4 *)(param_1 + 0x20) = 0;
  ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
            (5,*(undefined4 *)(param_1 + 0xc),0xffffffff8236ffbc);
  iVar5 = ?ReadRemoveVerifyProtocolVersion@CRequestObject@MassiveAdClient3@@IAAHXZ(param_1);
  if (iVar5 != 0) {
    cVar10 = ?ReadU8@CRequestObject@MassiveAdClient3@@IAAEXZ(param_1);
    ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
              (5,*(undefined4 *)(param_1 + 0xc),0xffffffff8236fa78,cVar10);
    if (cVar10 == -0x36) {
      uVar2 = ?ReadU32@CRequestObject@MassiveAdClient3@@IAAIXZ(param_1);
      ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
                (5,*(undefined4 *)(param_1 + 0xc),0xffffffff8236fa28,uVar2);
      uVar1 = *(uint *)(param_1 + 0x20);
      if ((uVar2 & 0xffffffff) != 0) {
        do {
          bVar11 = ?ReadU8@CRequestObject@MassiveAdClient3@@IAAEXZ(param_1);
          if (bVar11 < 0x49) {
            if (bVar11 == 0x48) {
              lVar3 = ?ReadString@CRequestObject@MassiveAdClient3@@IAAPADXZ(param_1);
              if (lVar3 == 0) {
LAB_834b1bfc:
                uVar4 = 0xffffffff8236fda8;
                goto LAB_834b1c4c;
              }
              pvVar6 = ??2CMassiveBaseObject@MassiveAdClient3@@SAPAXI@Z(0x1c);
              if (pvVar6 == (void *)0x0) {
                iVar5 = 0;
              }
              else {
                iVar5 = ??0CAddressIndexPair@MassiveAdClient3@@QAA@PADE@Z(pvVar6,lVar3,3);
              }
              if (iVar5 == 0) {
                uVar4 = 0xffffffff8236fcf0;
                goto LAB_834b1c4c;
              }
              pvVar6 = ??2CMassiveListNode@MassiveAdClient3@@SAPAXI@Z(0xc);
              if (pvVar6 == (void *)0x0) {
                uVar4 = 0;
              }
              else {
                uVar4 = ??0CMassiveListNode@MassiveAdClient3@@QAA@PAX@Z(pvVar6,iVar5);
              }
              ?Append@CMassiveList@MassiveAdClient3@@QAAHPAVCMassiveListNode@2@@Z
                        (param_1 + 0x50,uVar4);
              uVar4 = 0xffffffff8236fe90;
            }
            else {
              if (bVar11 == 0xd) {
                bVar11 = ?ReadU8@CRequestObject@MassiveAdClient3@@IAAEXZ(param_1);
                uVar4 = 0xffffffff8236ff20;
                *(undefined4 *)(param_1 + 0x7c) = 1;
                uVar8 = (ushort)bVar11;
                *(byte *)(param_1 + 0x7a) = bVar11;
                goto LAB_834b1b88;
              }
              if (bVar11 == 0x22) {
                lVar3 = ?ReadString@CRequestObject@MassiveAdClient3@@IAAPADXZ(param_1);
                if (lVar3 == 0) goto LAB_834b1bfc;
                pvVar6 = ??2CMassiveBaseObject@MassiveAdClient3@@SAPAXI@Z(0x1c);
                if (pvVar6 == (void *)0x0) {
                  iVar5 = 0;
                }
                else {
                  iVar5 = ??0CAddressIndexPair@MassiveAdClient3@@QAA@PADE@Z(pvVar6,lVar3,5);
                }
                if (iVar5 == 0) {
                  uVar4 = 0xffffffff8236fd48;
                  goto LAB_834b1c4c;
                }
                pvVar6 = ??2CMassiveListNode@MassiveAdClient3@@SAPAXI@Z(0xc);
                if (pvVar6 == (void *)0x0) {
                  uVar4 = 0;
                }
                else {
                  uVar4 = ??0CMassiveListNode@MassiveAdClient3@@QAA@PAX@Z(pvVar6,iVar5);
                }
                ?Append@CMassiveList@MassiveAdClient3@@QAAHPAVCMassiveListNode@2@@Z
                          (param_1 + 0x50,uVar4);
                uVar4 = 0xffffffff8236fec0;
              }
              else {
                if (bVar11 != 0x2d) {
                  if (bVar11 != 0x3a) {
                    if (bVar11 != 0x3b) goto LAB_834b19e0;
                    uVar4 = ?ReadU64@CRequestObject@MassiveAdClient3@@IAA_KXZ(param_1);
                    *(undefined8 *)(param_1 + 0x70) = uVar4;
                    ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
                              (5,*(undefined4 *)(param_1 + 0xc),0xffffffff8236fee0);
                    local_a0 = local_a0 + '\x01';
                    goto LAB_834b1b94;
                  }
                  uVar8 = ?ReadU16@CRequestObject@MassiveAdClient3@@IAAGXZ(param_1);
                  *(ushort *)(param_1 + 0x78) = uVar8;
                  uVar4 = 0xffffffff8236fef8;
                  goto LAB_834b1b88;
                }
                lVar3 = ?ReadString@CRequestObject@MassiveAdClient3@@IAAPADXZ(param_1);
                if (lVar3 == 0) {
                  uVar4 = 0xffffffff8236fe48;
                  goto LAB_834b1c4c;
                }
                pvVar6 = ??2CMassiveBaseObject@MassiveAdClient3@@SAPAXI@Z(0x1c);
                if (pvVar6 == (void *)0x0) {
                  iVar5 = 0;
                }
                else {
                  iVar5 = ??0CAddressIndexPair@MassiveAdClient3@@QAA@PADE@Z(pvVar6,lVar3,4);
                }
                if (iVar5 == 0) {
                  uVar4 = 0xffffffff8236fdf0;
                  goto LAB_834b1c4c;
                }
                pvVar6 = ??2CMassiveListNode@MassiveAdClient3@@SAPAXI@Z(0xc);
                if (pvVar6 == (void *)0x0) {
                  uVar4 = 0;
                }
                else {
                  uVar4 = ??0CMassiveListNode@MassiveAdClient3@@QAA@PAX@Z(pvVar6,iVar5);
                }
                ?Append@CMassiveList@MassiveAdClient3@@QAAHPAVCMassiveListNode@2@@Z
                          (param_1 + 0x50,uVar4);
                uVar4 = 0xffffffff8236fea8;
              }
            }
            ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
                      (5,*(undefined4 *)(param_1 + 0xc),uVar4,*(undefined4 *)(iVar5 + 0x18));
            (*(code *)PTR_free_84494774)(lVar3);
          }
          else if (bVar11 == 0x49) {
            puVar7 = (undefined4 *)??2CMassiveBaseObject@MassiveAdClient3@@SAPAXI@Z(0x18);
            if (puVar7 == (undefined4 *)0x0) {
              puVar7 = (undefined4 *)0x0;
            }
            else {
              uVar9 = ?ReadU16@CRequestObject@MassiveAdClient3@@IAAGXZ(param_1);
              ??0CMassiveBaseObject@MassiveAdClient3@@QAA@PAD@Z(puVar7,0xffffffff8236fb48);
              *puVar7 = &PTR_??_GCPortIndexPair@MassiveAdClient3@@UAAPAXI@Z_8236fb44;
              *(undefined2 *)((int)puVar7 + 0x16) = uVar9;
              *(undefined *)(puVar7 + 5) = 3;
            }
            if (puVar7 == (undefined4 *)0x0) {
              uVar4 = 0xffffffff8236fbf8;
              goto LAB_834b1c4c;
            }
            pvVar6 = ??2CMassiveListNode@MassiveAdClient3@@SAPAXI@Z(0xc);
            if (pvVar6 == (void *)0x0) {
              uVar4 = 0;
            }
            else {
              uVar4 = ??0CMassiveListNode@MassiveAdClient3@@QAA@PAX@Z(pvVar6,puVar7);
            }
            ?Append@CMassiveList@MassiveAdClient3@@QAAHPAVCMassiveListNode@2@@Z
                      (param_1 + 0x60,uVar4);
            uVar4 = 0xffffffff8236ff5c;
LAB_834b1b84:
            uVar8 = *(ushort *)((int)puVar7 + 0x16);
LAB_834b1b88:
            ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
                      (5,*(undefined4 *)(param_1 + 0xc),uVar4,uVar8);
          }
          else {
            if (bVar11 == 0x4a) {
              puVar7 = (undefined4 *)??2CMassiveBaseObject@MassiveAdClient3@@SAPAXI@Z(0x18);
              if (puVar7 == (undefined4 *)0x0) {
                puVar7 = (undefined4 *)0x0;
              }
              else {
                uVar9 = ?ReadU16@CRequestObject@MassiveAdClient3@@IAAGXZ(param_1);
                ??0CMassiveBaseObject@MassiveAdClient3@@QAA@PAD@Z(puVar7,0xffffffff8236fb48);
                *puVar7 = &PTR_??_GCPortIndexPair@MassiveAdClient3@@UAAPAXI@Z_8236fb44;
                *(undefined2 *)((int)puVar7 + 0x16) = uVar9;
                *(undefined *)(puVar7 + 5) = 4;
              }
              if (puVar7 != (undefined4 *)0x0) {
                pvVar6 = ??2CMassiveListNode@MassiveAdClient3@@SAPAXI@Z(0xc);
                if (pvVar6 == (void *)0x0) {
                  uVar4 = 0;
                }
                else {
                  uVar4 = ??0CMassiveListNode@MassiveAdClient3@@QAA@PAX@Z(pvVar6,puVar7);
                }
                ?Append@CMassiveList@MassiveAdClient3@@QAAHPAVCMassiveListNode@2@@Z
                          (param_1 + 0x60,uVar4);
                uVar4 = 0xffffffff8236ff78;
                goto LAB_834b1b84;
              }
              uVar4 = 0xffffffff8236fc48;
              goto LAB_834b1c4c;
            }
            if (bVar11 == 0x4b) {
              puVar7 = (undefined4 *)??2CMassiveBaseObject@MassiveAdClient3@@SAPAXI@Z(0x18);
              if (puVar7 == (undefined4 *)0x0) {
                puVar7 = (undefined4 *)0x0;
              }
              else {
                uVar9 = ?ReadU16@CRequestObject@MassiveAdClient3@@IAAGXZ(param_1);
                ??0CMassiveBaseObject@MassiveAdClient3@@QAA@PAD@Z(puVar7,0xffffffff8236fb48);
                *puVar7 = &PTR_??_GCPortIndexPair@MassiveAdClient3@@UAAPAXI@Z_8236fb44;
                *(undefined2 *)((int)puVar7 + 0x16) = uVar9;
                *(undefined *)(puVar7 + 5) = 5;
              }
              if (puVar7 != (undefined4 *)0x0) {
                pvVar6 = ??2CMassiveListNode@MassiveAdClient3@@SAPAXI@Z(0xc);
                if (pvVar6 == (void *)0x0) {
                  uVar4 = 0;
                }
                else {
                  uVar4 = ??0CMassiveListNode@MassiveAdClient3@@QAA@PAX@Z(pvVar6,puVar7);
                }
                ?Append@CMassiveList@MassiveAdClient3@@QAAHPAVCMassiveListNode@2@@Z
                          (param_1 + 0x60,uVar4);
                uVar4 = 0xffffffff8236ff98;
                goto LAB_834b1b84;
              }
              uVar4 = 0xffffffff8236fc98;
              goto LAB_834b1c4c;
            }
            if (bVar11 == 0x54) {
              bVar11 = ?ReadU8@CRequestObject@MassiveAdClient3@@IAAEXZ(param_1);
              uVar4 = 0xffffffff8236ff40;
              *(undefined4 *)(param_1 + 0x84) = 1;
              uVar8 = (ushort)bVar11;
              *(byte *)(param_1 + 0x80) = bVar11;
              goto LAB_834b1b88;
            }
LAB_834b19e0:
            iVar5 = ?SkipField@CRequestObject@MassiveAdClient3@@IAAHE@Z(param_1);
            if (iVar5 == 0) goto LAB_834b1c58;
          }
LAB_834b1b94:
        } while (((ulonglong)*(uint *)(param_1 + 0x20) - (ulonglong)uVar1 & 0xffffffff) <
                 (uVar2 & 0xffffffff));
      }
      if (local_a0 == '\x01') {
        ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
                  (5,*(undefined4 *)(param_1 + 0xc),0xffffffff8236f9e0);
        ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
                  (5,*(undefined4 *)(param_1 + 0xc),0xffffffff8236f9b8);
        uVar4 = 1;
        goto LAB_834b1c5c;
      }
      uVar4 = 0xffffffff8236f980;
LAB_834b1c4c:
      ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
                (2,*(undefined4 *)(param_1 + 0xc),uVar4);
    }
    else {
      ?Log@CLog@MassiveAdClient3@@SAXW4__MASSIVE_ENUM_LOG_LEVEL@2@PAD1ZZ
                (2,*(undefined4 *)(param_1 + 0xc),0xffffffff8236fa3c,cVar10);
    }
  }
LAB_834b1c58:
  uVar4 = 0;
LAB_834b1c5c:
  __restgprlr(uVar4);
  return;
}
```

</details>