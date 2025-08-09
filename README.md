## Edge-TTS: Text-to-Speech on WSL/Ubuntu
A step-by-step guide to installing and using Edge-TTS, Microsoft’s Neural Text-to-Speech engine, with custom voices, rate/pitch control, and support for long text.

## Features
100+ Microsoft Neural voices, 
Control rate (speed) and pitch, 
Read from text files or inline text, 
Save output MP3s directly to Windows folders, 
Works inside WSL/Ubuntu, 
Fully offline virtual environment (no system pollution)

## Requirements
Windows with WSL installed (Ubuntu recommended)
Python 3.9 or later inside WSL
Internet connection (voices are cloud-based)

## Installation
## 1) Install dependencies
```
sudo apt update && sudo apt install -y python3 python3-venv python3-pip
```

## 2) Create a project folder and virtual environment
```
mkdir -p ~/tts
cd ~/tts
python3 -m venv .venv
source .venv/bin/activate
```

## 3) Install Edge-TTS & Rich
```
pip install --upgrade pip
pip install edge-tts rich
```

## 4) Create the tool file
Run:
```
nano edge_tts_tool.py
```
Paste:
```
import asyncio
import argparse
import os
import sys
from pathlib import Path
from rich import print, box
from rich.console import Console
from rich.table import Table
import edge_tts

console = Console()

async def list_voices(filter_str=None, limit=None):
    voices = await edge_tts.list_voices()
    rows = []
    for v in voices:
        name = v.get("ShortName", "")
        locale = v.get("Locale", "")
        gender = v.get("Gender", "")
        style = ", ".join(v.get("StyleList", []) or [])
        if filter_str and filter_str.lower() not in name.lower() and filter_str.lower() not in locale.lower():
            continue
        rows.append((name, locale, gender, style))

    if not rows:
        console.print("[yellow]No voices matched your filter.[/yellow]")
        return

    table = Table(title="Edge TTS Voices", show_lines=False, box=box.SIMPLE_HEAVY)
    table.add_column("ShortName", style="cyan", no_wrap=True)
    table.add_column("Locale", style="magenta", no_wrap=True)
    table.add_column("Gender", style="green", no_wrap=True)
    table.add_column("Styles", style="white")

    count = 0
    for r in rows:
        table.add_row(*r)
        count += 1
        if limit and count >= limit:
            break

    console.print(table)
    console.print(f"[bold]Total shown:[/bold] {min(count, len(rows))}  [bold]Available:[/bold] {len(rows)}")

async def synth(text, voice="en-US-AriaNeural", outfile="output.mp3", rate="+0%", pitch="+0Hz"):
    communicate = edge_tts.Communicate(text=text, voice=voice, rate=rate, pitch=pitch)
    console.print(f"[bold]Generating:[/bold] '{outfile}' with [cyan]{voice}[/cyan] (rate={rate}, pitch={pitch})")
    await communicate.save(outfile)
    console.print(f"[green]Saved:[/green] {outfile}")

def read_text_input(text=None, infile=None):
    if text:
        return text
    if infile:
        p = Path(infile)
        if not p.exists():
            console.print(f"[red]Input file not found:[/red] {infile}")
            sys.exit(1)
        return p.read_text(encoding="utf-8")
    if not sys.stdin.isatty():
        return sys.stdin.read()
    console.print("[red]No text provided. Use --text, --infile, or pipe text in.[/red]")
    sys.exit(1)

def main():
    parser = argparse.ArgumentParser(description="Edge TTS helper")
    sub = parser.add_subparsers(dest="cmd")

    ls = sub.add_parser("list", help="List voices")
    ls.add_argument("--filter", help="Filter by substring in ShortName or Locale")
    ls.add_argument("--limit", type=int, help="Limit number shown")

    sx = sub.add_parser("say", help="Synthesize speech")
    sx.add_argument("--voice", default="en-US-AriaNeural", help="Voice ShortName")
    sx.add_argument("--text", help="Text to speak")
    sx.add_argument("--infile", help="Path to UTF-8 text file")
    sx.add_argument("--outfile", default="output.mp3", help="Output audio file (.mp3 recommended)")
    sx.add_argument("--rate", default="+0%", help="Speech rate, e.g. +10% or -10%")
    sx.add_argument("--pitch", default="+0Hz", help="Pitch, e.g. +2Hz or -2Hz")

    args = parser.parse_args()

    if args.cmd == "list":
        asyncio.run(list_voices(filter_str=args.filter, limit=args.limit))
    elif args.cmd == "say":
        text = read_text_input(args.text, args.infile)
        asyncio.run(synth(text, args.voice, args.outfile, args.rate, args.pitch))
        try:
            if sys.platform.startswith("win"):
                os.startfile(args.outfile)  # type: ignore
            elif sys.platform == "darwin":
                os.system(f"open '{args.outfile}'")
            else:
                os.system(f"xdg-open '{args.outfile}'")
        except Exception:
            pass
    else:
        parser.print_help()

if __name__ == "__main__":
    main()
```
Press Ctrl+X, then Y, Then press enter to save 

## 5) List voices
```
 python3 edge_tts_tool.py list --limit 20
```
List English Voices
```
python3 edge_tts_tool.py list --filter en-
```

## 6) Now Try it out
```
python3 edge_tts_tool.py say --voice en-US-GuyNeural --text "Hello from Edge TTS" --outfile test.mp3
```

## 7) Make the file open with windows Player
```
explorer.exe "$(wslpath -w test.mp3)"
```

## 8) Play (Ubuntu/WSL can use mpv, vlc, or play)
```
sudo apt install -y mpg123
mpg123 test.mp3
```

## 9)  Make the Audio File download on windows, and also add Rate and Pitch (How fast/slow and how high you want the audio to be)
A) For it to download to window
```
nano edge_tts_tool.py
```
Find this 
```
sx.add_argument("--outfile", default="output.mp3", help="Output audio file (.mp3 recommended)")
```
Replace it with this
```
sx.add_argument("--outfile", default="/mnt/c/Users/MSI/Downloads/output.mp3", help="Output audio file (.mp3 recommended)")
```
Press Ctrl+X, then Y, Then press enter to save

B) Add Rate and Pitch
```
nano edge_tts_tool.py
```
Find this 
```
sx.add_argument("--rate", default="+0%", help="Speech rate, e.g. +10% or -10%")
sx.add_argument("--pitch", default="+0Hz", help="Pitch, e.g. +2Hz or -2Hz")
```
Replace it with this
```
sx.add_argument("--rate", type=str, default="+0%", help="Speech rate, e.g. +10% or -10%")
sx.add_argument("--pitch", type=str, default="+0Hz", help="Pitch, e.g. +2Hz or -2Hz")
```
Press Ctrl+X, then Y, Then press enter to save

## 10) To find your "UserProfile"/Windows Name
Open Command prompt, type this code
```
whoami
```

## 11) Lets Try out a Text
```
python3 edge_tts_tool.py say \
  --voice <VoiceName> \
  --text "Your text here" \
  --rate="rate%" \
  --pitch="pitchHz"
```
Where you would replace <Voice> with the voice you want to you for the speech, same with Text, for rate and pitch, it should be like the --rate="+5%" \ or --pitch="-60Hz"

## 12) Creating a Long Text to speech
Open Nano:
```
nano myscript.txt
```

# Type or paste your long form text  
Press Ctrl+X, then Y, Then press enter to save

Now, lets run it
```
python3 edge_tts_tool.py say \
--voice <VoiceName> \
--rate="rate%" \
--pitch="pitchHz" \
--infile myscript.txt \
--outfile /mnt/c/Users/USERPROFILE/Downloads/myspeech.mp3
```
Remember to always rename your file after it gets downloaded, so it wont be replaced when you run a new one

## 13) How to Get back in after a restart
Enter Directory
```
cd ~/tts
```
Activate your virtual environment
```
source .venv/bin/activate
```
Run a script to be sure 
```
python3 edge_tts_tool.py list --filter en-
```
This to show English voice
OR
```
python3 edge_tts_tool.py say --voice en-US-GuyNeural --text "Back after restart"
```
To get this audio out.

## Disclaimer
This repository and guide are for educational purposes only.
Microsoft Edge TTS is an online service provided by Microsoft, and its availability, supported voices, and usage limits are subject to change at any time by Microsoft.
Users are responsible for ensuring their use complies with all applicable laws, regulations, and Microsoft’s terms of service.
The author of this guide is not affiliated with Microsoft and assumes no liability for how this software or service is used.
