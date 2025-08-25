# Massive Ad Client [DRAFT]

23/08/25 - very early version

---

## INTRO

## HISTORY

## Known Versions

At this point in time, thanks to website "dlldownloader" and a lot of games I was able to get in my possession 4 out of 5 known versions of the MAD:

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

If we look though the function flow, it's rather simple.

- Check first byte for version value (3 in our case with NFS:Carbon)
- Check reply task id to be equal `0xCA`
- Read u32 of packet size
- Parse fields with values of `0x48`, `0x22`, `0x2d`, `0x49`, `0x4A`, `0x4B`, `0x3A`, `0x3B`, `0x0D` and `0x54`
- Return from the function

Not all values are mandatory, and since we have names for all of them from Sk8te 2 symbols we can construct this enum

```rust
#[repr(u8)]
pub enum LocationResponseMADFields {
    ZoneServerUrl=0x48,
    ZoneServerPort=0x49,
    MediaServerUrl=0x2d,
    MediaServerPort=0x4a,
    ImpressionsServerUrl=0x22,
    ImpressionsServerPort=0x4b,
    HeartBeatPeriod=0x54,
    ServerConfigurableOptions=0x3a,
    ReadingServerTime=0x3b

}
```

By checking logic in `ReadU8`, `ReadString` and others we can construct a rather simple reply writer of:

```rust
#[derive(Debug, Eq, PartialEq, TryFromPrimitive, IntoPrimitive)]
#[repr(u8)]
pub enum MadServerProtocolVersion {
    Version1 = 0x01,
    Version2 = 0x02,
    Version3 = 0x03,
    Version4 = 0x04,
}

#[derive(Debug, Eq, PartialEq, TryFromPrimitive, IntoPrimitive)]
#[repr(u8)]
pub enum MadReplyTaskId {
    LocateService = 0xCA,
}

pub struct MADReply {
    version: MadServerProtocolVersion,
    task_id: MadReplyTaskId,
    buf: Vec<u8>,
}

impl MADReply {
    pub fn new(protocol_version: MadServerProtocolVersion, task_id: MadReplyTaskId) -> Self {
        MADReply {
            version: protocol_version,
            task_id: task_id,
            buf: Vec::new(),
        }
    }

    pub fn add_string<E: Into<u8>>(&mut self, field: E, value: &str) {
        let data = value.as_bytes();
        self.buf.push(field.into());
        self.buf
            .write_u16::<LittleEndian>(data.len() as u16)
            .unwrap();
        self.buf.extend_from_slice(data);
    }

    pub fn add_u8<E: Into<u8>>(&mut self, field: E, value: u8) {
        self.buf.push(field.into());
        self.buf.write_u8(value).unwrap();
    }

    pub fn add_u16<E: Into<u8>>(&mut self, field: E, value: u16) {
        self.buf.push(field.into());
        self.buf.write_u16::<LittleEndian>(value).unwrap();
    }

    pub fn add_u32(&mut self, field: LocationResponseMADFields, value: u32) {
        self.buf.push(field as u8);
        self.buf.write_u32::<LittleEndian>(value).unwrap();
    }

    pub fn add_u64(&mut self, field: LocationResponseMADFields, value: u64) {
        self.buf.push(field as u8);
        self.buf.write_u64::<LittleEndian>(value).unwrap();
    }

    pub fn into_bytes(self) -> Vec<u8> {
        self.buf
    }

    pub fn build_reply(self) -> Vec<u8> {

        let mut final_buf: Vec<u8> = Vec::new();
        final_buf.push(self.version.into());
        final_buf.push(self.task_id.into());
        final_buf.write_u32::<LittleEndian>(self.buf.len() as u32);
        final_buf.extend_from_slice(&self.buf);

        final_buf
    }
}
```

Which we utilize as

```rust
    let mut reply = MADReply::new(
        MadServerProtocolVersion::Version3,
        MadReplyTaskId::LocateService,
    );

    reply.add_string(LocationResponseMADFields::ZoneServerUrl, "http://127.0.0.1");
    reply.add_u16(LocationResponseMADFields::ZoneServerPort, 80);
    reply.add_string(
        LocationResponseMADFields::MediaServerUrl,
        "http://127.0.0.1",
    );
    reply.add_u16(LocationResponseMADFields::MediaServerPort, 80);
    reply.add_string(
        LocationResponseMADFields::ImpressionsServerUrl,
        "http://127.0.0.1",
    );
    reply.add_u16(LocationResponseMADFields::ImpressionsServerPort, 80);
    reply.add_u32(LocationResponseMADFields::HeartBeatPeriod, 60);
    reply.add_u64(
        LocationResponseMADFields::ReadingServerTime,
        SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs(),
    );

    let bytes = reply.build_reply();


    HttpResponse::Ok()
    .content_type("application/massive")
    .body(bytes)
```

We launch the game, see the request, send the reply and then...

**Nothing**

Okay, time to set a breakpoint in the function in NFS:Carbon and see where we fail exactly.

And we find out that function to parse the reply is not being called.

## Figuring out what are we missing

After some snooping in the binary we find the function which looks like a http reply header parser at `FUN_00873504`

```C++
undefined4 __fastcall FUN_00873504(int param_1)

{
  char *pcVar1;
  char *pcVar2;
  int iVar3;
  undefined4 *puVar4;
  undefined local_108;
  undefined4 local_107;
  int local_8;
  
  if (*(int *)(param_1 + 0x30) == 0) {
    return 0;
  }
  if (*(char **)(param_1 + 0x20) == (char *)0x0) {
    return 0;
  }
  pcVar1 = strtok(*(char **)(param_1 + 0x20),&DAT_009e9b80);
  if (pcVar1 == (char *)0x0) {
    return 0;
  }
  local_8 = 0;
  local_108 = 0;
  puVar4 = &local_107;
  for (iVar3 = 0x3f; iVar3 != 0; iVar3 = iVar3 + -1) {
    *puVar4 = 0;
    puVar4 = puVar4 + 1;
  }
  *(undefined2 *)puVar4 = 0;
  *(undefined *)((int)puVar4 + 2) = 0;
  sscanf(pcVar1,s_%*s_%d_%256[_a-zA-Z0-9]_009fe964,&local_8,&local_108);
  if (local_8 == 200) {
    *(undefined4 *)(param_1 + 8) = 1;
    while (pcVar1 = strtok((char *)0x0,&DAT_009e9b80), pcVar1 != (char *)0x0) {
      pcVar2 = strstr(pcVar1,s_Content-Length:_009fe954);
      if (pcVar2 != (char *)0x0) {
        sscanf(pcVar1,s_%*s_%d_009fe94c,param_1 + 0x40);
      }
      pcVar1 = strstr(pcVar1,s_Transfer-Encoding:_chunked_009fe930);
      if (pcVar1 != (char *)0x0) {
        *(undefined4 *)(param_1 + 0x34) = 1;
      }
    }
    return 1;
  }
  if (local_8 < 400) {
    if (local_8 < 500) goto LAB_0087361c;
  }
  else if (local_8 < 500) {
    *(undefined4 *)(param_1 + 8) = 1;
    do {
      pcVar1 = strtok((char *)0x0,&DAT_009e9b80);
    } while (pcVar1 != (char *)0x0);
    return 0;
  }
  if (local_8 < 600) {
    *(undefined4 *)(param_1 + 8) = 1;
    do {
      pcVar1 = strtok((char *)0x0,&DAT_009e9b80);
    } while (pcVar1 != (char *)0x0);
    return 0;
  }
LAB_0087361c:
  *(undefined4 *)(param_1 + 8) = 1;
  return 0;
}
```

Let's try to set breakpoint in it and check step by step where are we failing exactly. And after one long long run though all op codes we realize that issue is not a missing header, but that the MassiveAdClient lib expecte camel case headers, instead of default lowercased once.

With a simple fix of:

```rust
    let mut reply = HttpResponse::Ok().body(bytes);    
    reply.head_mut().set_camel_case_headers(true);
    reply
```

we're now hitting the reply parser!

Sadly, this didn't trigger the game to request more data, so, we're back to the debugger.

## Debugging parser issues

This one took a while to figure out, generally to my own stupidity.

So! To leave the loop with a value of 1 we need to have a flag of `if_all_required_variables_were_parsed` of 1

```C++
    if (message_size <= (uint)(param_1->field32_0x20 - iVar1)) {
      if (if_all_required_variables_were_parsed != '\x01') {
        return 0;
      }
      return 1;
    }
```

which is set here

```C++
          else if (field_id_2 == 0x3b) {
            server_time = ReadU64((u_long)param_1);
            if_all_required_variables_were_parsed = if_all_required_variables_were_parsed + '\x01';
            param_1->server_time = server_time;
          }
```

So, we could trigger a "good ending" with only sending a packet consisting of field `0x3b` and U64 following it. Which again, didn't work. During that time I also fixed some bugs, like, lenght of strings is defgined as big endianess, not little.

```rust
    pub fn add_string<E: Into<u8>>(&mut self, field: E, value: &str) {
        let data = value.as_bytes();
        self.buf.push(field.into());
        self.buf
            .write_u16::<BigEndian>(data.len() as u16)
            .unwrap();
        self.buf.extend_from_slice(data);
    }
```

After speding a lot of time trying to figure out what field exactly is malformed I **finally** decided to actually check if we're hitting this line

```C++
    if (message_size <= (uint)(param_1->field32_0x20 - iVar1)) {
      if (if_all_required_variables_were_parsed != '\x01') { //<< This one
        return 0;
      }
      return 1;
    }
```

And I find out that I don't.

That's when I decided to **actually** check the very simple check of `if (message_size <= (uint)(param_1->field32_0x20 - iVar1))` and to my suprise the data was written in big endiannes, instead of small endiannes I was sending.

```rust
    pub fn build_reply(self) -> Vec<u8> {

        let mut final_buf: Vec<u8> = Vec::new();
        final_buf.push(self.version.into());
        final_buf.push(self.task_id.into());
        final_buf.write_u32::<LittleEndian>(self.buf.len() as u32);
        final_buf.extend_from_slice(&self.buf);

        final_buf
    }
```

Well, 1 change of

```rust
final_buf.write_u32::<BigEndian>(self.buf.len() as u32);
```

And now comparasion works like it should and we exit with a result of 1. Well, that was dissapointingly dumb. Mainly in myself. But it seems like MassiveAdClient prefer big endiannes for numbers, so it should be easier from now on to deduct stuff.

And now we have a new request!

```
[2025-08-25T04:37:29Z INFO  actix_web::middleware::logger] 127.0.0.1 "POST /adsrv/locateService HTTP/1.1" 200 56 "-" "Adclient Massive Inc./3.2.1.55" 0.000963
[2025-08-25T04:37:48Z INFO  actix_web::middleware::logger] 127.0.0.1 "POST /adsrv/openSession HTTP/1.1" 404 0 "-" "Adclient Massive Inc./3.2.1.55" 0.000210
```

## OpenSession request

We check new request payload and

![alt text](img/open_session.png)

Oh. It's gonna take a while. But we do have symbols!

![alt text](img/open_session_symbols.png)

