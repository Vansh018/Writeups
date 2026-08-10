TRYHACKME — AFTER HOURS

This room is a part of THM’s HackerHolidays 2026.

🛎️ CONCIERGE BRIEFING

The challenge gives us a set of Windows system artifacts and tells us that something is maintaining persistence somewhere that normal persistence checks might miss.

The important part is:

“hiding somewhere quieter”

So instead of looking only at Startup, Scheduled Tasks, or Run keys, we need to inspect the provided system artifacts for custom configuration data.

🧳 ATTACHMENTS

The files were already present in the AttackBox:

/root/Rooms/hacker-holidays-2026/after-hours/after-hours-forensics-hh/challenge_attachments

First I checked what was inside:

file *

This gave:

INDEX.BTR
MAPPING1.MAP
MAPPING2.MAP
MAPPING3.MAP
OBJECTS.DATA

The files looked like Windows WMI repository artifacts.

OBJECTS.DATA was the interesting one.

🔎 LOOKING THROUGH OBJECTS.DATA

I started with strings:

strings -a OBJECTS.DATA | head -100

Most of the output was normal WMI/CIM information.

Then I searched for anything suspicious:

strings -a OBJECTS.DATA | grep -Ei '.exe|.dll|powershell|cmd|http|https|base64|assembly|namespace|class'

This revealed something very interesting:

CommandLineEventConsumer
cmd /C powershell.exe -Sta -Nop -Window Hidden -enc ...

So we had an encoded PowerShell command hidden inside the WMI repository.

The use of:

- CommandLineEventConsumer
- powershell.exe
- -Window Hidden
- -enc

was a pretty big indicator that this was the persistence mechanism.

🧩 FINDING THE CUSTOM CLASS

The output also contained:

Win32_HardwareTelemetry
ConfigData
string

Immediately after ConfigData was a very long Base64-looking string.

I searched for the class directly:

strings -a OBJECTS.DATA | grep -n -i -A5 -B5 "Win32_HardwareTelemetry"

And there it was.

Win32_HardwareTelemetry
ConfigData
string
7VZPbFRFGP/edillgUrBAJWA...

The important thing here was that the data wasn't simply readable text.

It was Base64 encoded.

📦 EXTRACTING THE DATA

I extracted the Base64 value using awk.

The first attempt was slightly off because the line after the class name was:

ConfigData
string
<base64>

So I needed to move three lines forward:

strings -a OBJECTS.DATA | awk '
/^Win32_HardwareTelemetry$/ {
    getline
    getline
    getline
    print
    exit
}' > config.b64

Then I checked it:

wc -c config.b64
head -c 80 config.b64
echo

This gave a Base64 string around 2.2 KB long.

I decoded it:

base64 -d config.b64 > payload.bin

Checking the resulting file:

ls -lh payload.bin
file payload.bin

It was just reported as:

payload.bin: data

So it wasn't a normal file format yet.

🗜️ DECOMPRESSING THE PAYLOAD

The PowerShell command we found earlier gave us the clue.

The script was doing:

[IO.Compression.DeflateStream]

and then reading the decoded data.

So the next step was to try raw DEFLATE decompression.

I used Python:

python3 - <<'PY'
import zlib

with open("payload.bin", "rb") as f:
    data = f.read()

for wbits in [-zlib.MAX_WBITS, zlib.MAX_WBITS]:
    try:
        out = zlib.decompress(data, wbits)
        print(f"Success with wbits={wbits}, size={len(out)}")
        open("payload.dll", "wb").write(out)
        break
    except zlib.error as e:
        print(f"Failed with wbits={wbits}: {e}")
PY

The important part was:

Success with wbits=-15, size=4096

That told us we had successfully decompressed the raw DEFLATE stream.

Now:

file payload.dll

returned:

PE32 executable (GUI) Intel 80386 Mono/.Net assembly, for MS Windows, 3 sections

And the header confirmed it was a PE:

xxd -l 32 payload.dll

4d 5a ...

MZ.

So we had successfully recovered the embedded .NET payload.

🔬 ANALYSING THE DLL

I first tried looking for an obvious flag:

strings -a payload.dll | grep -Ei 'THM|flag|HHC|HH|[A-Za-z0-9_]+\{'

Nothing useful came back.

I also searched for common encoding/decryption keywords:

strings -a payload.dll | grep -Ei 'base64|decode|decrypt|password|secret|key'

Again, nothing useful.

So I just dumped the strings from the payload and looked manually.

Eventually, something interesting appeared:

bytelotusdc
cmd.exe
/c net user patch VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9 /add

There was our flag.

The payload was executing:

cmd.exe /c net user patch <BASE64> /add

So it was creating a Windows user named:

patch

with the Base64 string as its password.

🚩 DECODING THE FLAG

The value was:

VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9

It is Base64.

Decoding it gives:

THM{P4tch_op3n3d_tH3_BacKd00r}

And that's the flag.

📝 FINAL FLAG

THM{P4tch_op3n3d_tH3_BacKd00r}

🧠 WHAT HAPPENED?

The persistence wasn't stored in the usual places.

Instead, the attacker created a custom WMI class:

Win32_HardwareTelemetry

with a property:

ConfigData

The property contained a Base64-encoded, raw-DEFLATE-compressed .NET executable.

The WMI repository also contained a CommandLineEventConsumer which launched PowerShell with:

-Window Hidden
-enc

The PowerShell script decoded/decompressed the payload and loaded the .NET assembly directly.

The recovered payload then executed:

cmd.exe /c net user patch <password> /add

The password itself was Base64 encoded and contained the final flag.

So the chain was basically:

WMI Repository
      ↓
Win32_HardwareTelemetry
      ↓
ConfigData
      ↓
Base64
      ↓
Raw DEFLATE
      ↓
.NET DLL
      ↓
net user patch
      ↓
Base64 password
      ↓
FLAG

And that's it.

Another HackerHolidays room done.