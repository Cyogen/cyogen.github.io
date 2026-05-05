+++
title = "Zero to HTB: My Lab, Tools, and Workflow"
date = "2026-05-05T00:00:00-05:00"
tags = ["htb", "setup", "methodology", "kali", "terminator", "cherrytree", "workflow", "bash", "tools"]
description = "A complete walkthrough of my HTB environment — Kali setup, Terminator config, a .bashrc notes system that puts your cheatsheets in the terminal, CherryTree note-taking workflow, and the methodology I use to approach every box."
draft = false
+++

This is not a tools list. You can find those anywhere. This is the actual setup — the specific config values, the exact keybindings, the small decisions that compound into a workflow that gets out of your way and lets you focus on the box. Start to finish.

---

## PT 1 — The Environment

### Kali Linux

Kali is the obvious choice and the right one. The tooling is pre-installed, the community uses it, and writeups are written against it. Running it as a VM on your host gives you snapshots, easy networking config, and the ability to reset if something goes sideways.

I run Kali in QEMU/KVM with virt-manager. VMware and VirtualBox both work fine — the preference is personal. The important things are:

- **Minimum 4 cores, 8GB RAM.** BloodHound with a full domain dump and a couple of browser tabs will eat a weaker setup.
- **Give it a real disk allocation, not dynamic.** Dynamic disks fragment badly under heavy I/O. 80GB fixed is a reasonable starting point.
- **Enable clipboard sharing and drag-and-drop.** Install `spice-vdagent` on the guest if it isn't there already. Being able to paste between host and VM is not optional.

### HTB VPN

Every active session starts here:

```bash
sudo openvpn ~/htb/lab_<username>.ovpn &
```

Verify you're connected:

```bash
ip a show tun0
```

Your `tun0` address is your attacker IP — the one you put in every reverse shell. Export it immediately:

```bash
export ATTACKER=$(ip a show tun0 | grep 'inet ' | awk '{print $2}' | cut -d/ -f1)
```

That lives in `.bashrc` so it auto-populates on every new shell. More on that later.

### Folder Structure

Every machine gets its own directory. Keep it consistent:

```
~/htb/
├── lab_<username>.ovpn
├── machines/
│   └── machinename/
│       ├── nmap/
│       ├── exploits/
│       ├── loot/
│       └── notes.ctb
```

Create it with one alias — also in `.bashrc`:

```bash
newbox() {
    mkdir -p ~/htb/machines/$1/{nmap,exploits,loot}
    cd ~/htb/machines/$1
    echo "[+] $1 ready"
}
```

Run `newbox shocker`, `cd ~/htb/machines/shocker`, and you're set.

---

## PT 2 — Terminator

Terminator is the terminal. Not because it's the flashiest option, but because split panes and keyboard-only navigation are non-negotiable for this work, and Terminator does both without requiring a tiling window manager config.

### Install

```bash
sudo apt install terminator -y
```

### Configuration

Terminator's config file lives at `~/.config/terminator/config`. Replace the default with this:

```ini
[global_config]
  title_transmit_bg_color = "#ff0000"
  title_receive_bg_color = "#1a1a1a"
  title_inactive_bg_color = "#1a1a1a"
  suppress_multiple_term_dialog = True
[keybindings]
  split_horiz = <Ctrl><Shift>o
  split_vert = <Ctrl><Shift>e
  close_term = <Ctrl><Shift>w
  go_next = <Ctrl>Tab
  zoom_terminal = <Ctrl><Shift>x
  search = <Ctrl><Shift>f
  copy = <Ctrl><Shift>c
  paste = <Ctrl><Shift>v
[profiles]
  [[default]]
    background_color = "#0d0d0d"
    foreground_color = "#e8e8e8"
    cursor_color = "#00ff41"
    font = Hack Nerd Font Mono 11
    use_system_font = False
    scrollback_lines = 10000
    scrollback_infinite = False
    scroll_on_output = False
    scroll_on_keystroke = True
    show_titlebar = False
    palette = "#1a1a1a:#ff5555:#50fa7b:#f1fa8c:#bd93f9:#ff79c6:#8be9fd:#f8f8f2:#44475a:#ff6e6e:#69ff94:#ffffa5:#d6acff:#ff92df:#a4ffff:#ffffff"
[layouts]
  [[default]]
    [[[child1]]]
      type = Terminal
      parent = window0
    [[[window0]]]
      type = Window
      parent = ""
      size = 1400, 900
[plugins]
```

What this gives you:

- **`#0d0d0d` background** — not pure black, which causes eye strain over long sessions. Just dark enough.
- **`#00ff41` cursor** — the classic matrix green. Trivial choice, but a bright cursor prevents losing your place mid-scroll.
- **10,000 line scrollback** — nmap full port scans and verbose tool output are long. You will want to scroll up.
- **No titlebar** — recovers vertical space. The pane borders make the layout clear enough.
- **Hack Nerd Font** — required for tool output that uses special characters (ligature support, box-drawing characters in tools like pwncat). Install with `sudo apt install fonts-hack`.

### The Split Pane Workflow

The standard layout for a box is four panes:

```
┌─────────────────────┬─────────────────┐
│                     │                 │
│   main shell        │   listener /    │
│   (enum, exploits)  │   shell catch   │
│                     │                 │
├─────────────────────┼─────────────────┤
│                     │                 │
│   file server       │   notes / misc  │
│   (http/smb)        │                 │
│                     │                 │
└─────────────────────┴─────────────────┘
```

Open Terminator, then:

| Action | Shortcut |
|--------|----------|
| Split horizontally | `Ctrl+Shift+O` |
| Split vertically | `Ctrl+Shift+E` |
| Navigate between panes | `Alt+Arrow` |
| Maximise current pane | `Ctrl+Shift+X` (toggle) |
| Close pane | `Ctrl+Shift+W` |
| Search scrollback | `Ctrl+Shift+F` |

`Ctrl+Shift+X` is the most used shortcut in the entire setup. When you need to read a long nmap output or a full `sudo -l` response, maximise the pane. When you're done, toggle back to the quad layout.

---

## PT 3 — .bashrc

The `.bashrc` is where the workflow lives. Not just aliases — functions that put your reference material directly in the terminal. No switching windows, no Googling syntax mid-engagement.

Open `~/.bashrc` and add the following blocks.

### The Notes System

The concept: a single `notes` function that takes a topic keyword and prints your cheatsheet directly in the terminal. Using `bat` for syntax-highlighted markdown rendering if it's installed, falling back to `cat`.

```bash
# ─── Notes ──────────────────────────────────────────────────────
notes() {
    local dir="$HOME/notes"
    local viewer="cat"
    command -v bat &>/dev/null && viewer="bat --language=md --style=plain"

    case "$1" in
        ad)     $viewer "$dir/active-directory.md" ;;
        privesc|lpe) $viewer "$dir/linux-privesc.md" ;;
        winpe)  $viewer "$dir/windows-privesc.md" ;;
        pivot)  $viewer "$dir/pivoting.md" ;;
        xfer)   $viewer "$dir/file-transfers.md" ;;
        crack)  $viewer "$dir/password-cracking.md" ;;
        web)    $viewer "$dir/web.md" ;;
        shells) $viewer "$dir/shells.md" ;;
        "")
            echo "Topics: ad  privesc  winpe  pivot  xfer  crack  web  shells"
            ;;
        *)
            echo "[-] Unknown topic: $1"
            echo "    Topics: ad  privesc  winpe  pivot  xfer  crack  web  shells"
            ;;
    esac
}
```

Install `bat`:

```bash
sudo apt install bat -y
# Kali aliases bat as batcat — fix it:
mkdir -p ~/.local/bin
ln -s /usr/bin/batcat ~/.local/bin/bat
```

Now `notes ad` prints your entire Active Directory cheatsheet with markdown syntax highlighting, right in the terminal. `notes privesc` pulls up your Linux privesc reference. No tab switching. No browser. The shorthand keywords are intentionally brief — `notes xfer` is faster to type under pressure than `notes file-transfers`.

Keep your notes in `~/notes/` as plain markdown files. The ones on this blog live there too.

### Target Management

```bash
# ─── HTB Target ─────────────────────────────────────────────────
export TARGET=""

htb() {
    case "$1" in
        set)
            export TARGET="$2"
            echo "[+] TARGET=$TARGET"
            ;;
        ip)
            echo "$TARGET"
            ;;
        url)
            echo "http://$TARGET"
            ;;
        *)
            echo "Usage: htb [set <ip>|ip|url]"
            ;;
    esac
}
```

Usage:

```bash
htb set 10.10.10.56
htb ip          # prints 10.10.10.56
htb url         # prints http://10.10.10.56
```

Every command that needs the target IP uses `$TARGET` rather than typing it out. Less typos, faster iteration.

### Attacker IP

```bash
# ─── Attacker IP (HTB VPN) ───────────────────────────────────────
export ATTACKER=""

update_attacker_ip() {
    ATTACKER=$(ip a show tun0 2>/dev/null | grep 'inet ' | awk '{print $2}' | cut -d/ -f1)
    [ -n "$ATTACKER" ] && echo "[+] ATTACKER=$ATTACKER"
}

alias vpnip='update_attacker_ip'
```

Run `vpnip` after connecting to HTB VPN. `$ATTACKER` is now populated for use in reverse shell one-liners.

### Scan Aliases

```bash
# ─── Nmap ────────────────────────────────────────────────────────
alias quickscan='nmap -sC -sV -oN nmap/quick.nmap $TARGET'
alias fullscan='nmap -p- -sC -sV --min-rate 5000 -oN nmap/full.nmap $TARGET'
alias udpscan='sudo nmap -sU --top-ports 100 -oN nmap/udp.nmap $TARGET'
```

`quickscan` first, always. `fullscan` in the background while you work the quick results.

### Servers

```bash
# ─── Servers ─────────────────────────────────────────────────────
alias pyhttp='python3 -m http.server 80'
alias pyhttp8='python3 -m http.server 8000'
alias smb='impacket-smbserver share . -smb2support'
alias smbauth='impacket-smbserver share . -smb2support -username user -password pass'
alias upload='python3 -m uploadserver 8000'
alias ftp='python3 -m pyftpdlib -p 21 -w'
```

### Listeners

```bash
# ─── Listeners ───────────────────────────────────────────────────
alias l443='rlwrap nc -lvnp 443'
alias l4444='rlwrap nc -lvnp 4444'
alias l80='rlwrap nc -lvnp 80'
```

`rlwrap` wraps the netcat listener with readline support — arrow keys, history, tab completion inside the reverse shell. Without it, up arrow prints `^[[A` into the shell. Install with `sudo apt install rlwrap -y`.

### Box Setup

```bash
# ─── Box Setup ───────────────────────────────────────────────────
newbox() {
    if [ -z "$1" ]; then
        echo "Usage: newbox <machinename>"
        return 1
    fi
    mkdir -p ~/htb/machines/$1/{nmap,exploits,loot}
    cd ~/htb/machines/$1
    echo "[+] Created and moved to ~/htb/machines/$1"
}
```

After all additions, reload:

```bash
source ~/.bashrc
```

---

## PT 4 — CherryTree

CherryTree is the notes application. Not Obsidian, not Notion, not a text file. CherryTree. The reasons:

- Works completely offline
- Rich text and code nodes in the same tree
- One `.ctb` file per machine — self-contained and portable
- Keyboard-driven enough to stay out of the way

```bash
sudo apt install cherrytree -y
```

### The Node Structure

Every machine gets this structure:

```
📁 MachineName
├── 📄 Machine Info
├── 📄 Nmap
├── 📄 Enumeration
├── 📄 Foothold
├── 📄 Lateral Movement
├── 📄 Privilege Escalation
├── 📄 Loot
│   ├── 📄 Credentials
│   └── 📄 Flags
└── 📄 Notes / Rabbit Holes
```

Create the top-level node with the machine name. Everything below it is a child node. One `.ctb` file per machine, saved in `~/htb/machines/<name>/notes.ctb`.

### Always Paste as Plain Text

This is the single most important CherryTree habit. Copying from a browser, a PDF, or another document brings formatting — fonts, colors, sizes — that pollute your notes and make them inconsistent.

**Never use `Ctrl+V`.**

**Always use `Ctrl+Shift+V`** to paste as plain text. Every time, without exception. Make it muscle memory before anything else in this section.

### Code Nodes — Black Background

For anything that is terminal output, command sequences, or code: right-click the parent node → **Add Child Node** → set the node type to **Code** and choose the appropriate syntax (bash, python, plain text). Code nodes render in monospace with a dark background automatically.

For inline command sections inside a rich text node:

1. Type or paste your command (plain text, `Ctrl+Shift+V`)
2. Select the text
3. Format → **Background Color** → set to `#000000`
4. Set foreground to `#e8e8e8` or `#00ff41`

This makes command blocks visually distinct from prose without needing a dedicated code node for every snippet.

### Highlighting — Red for Credentials and Flags

Credentials and flags get highlighted red. This is not decorative — when you are scrolling through three pages of notes looking for a password you found two hours ago, red text stops your eye immediately.

1. Select the text (a hash, a password, a flag value)
2. `Ctrl+H` → Highlight Color → choose red (`#ff0000` or a slightly softer `#cc0000`)

Everything else can be whatever. Credentials and flags are always red.

### Keyboard Navigation

Staying on the keyboard keeps the flow. The shortcuts that matter:

| Action | Shortcut |
|--------|----------|
| Move cursor word by word | `Ctrl+Left` / `Ctrl+Right` |
| Select word by word | `Ctrl+Shift+Left` / `Ctrl+Shift+Right` |
| Jump to top / bottom of node | `Ctrl+Home` / `Ctrl+End` |
| Jump to line start / end | `Home` / `End` |
| Find in current node | `Ctrl+F` |
| Find across all nodes | `Ctrl+Shift+F` |
| Rename current node | `F2` |
| Add child node | `Ctrl+Shift+N` |
| Toggle bold | `Ctrl+B` |
| Toggle monospace | `Ctrl+M` |

`Ctrl+Left` and `Ctrl+Right` are the most used. They let you move through a command or a hash word by word instead of holding an arrow key. Combined with `Shift`, you can select exactly the token you want without reaching for a mouse.

### The Master Template

Create this structure once, then **File → Import** it as a template for every new machine. Save it as `~/htb/template.ctb`.

```
📁 MACHINE_NAME
├── 📄 Machine Info
│       Name:
│       IP:
│       OS:
│       Difficulty:
│       Released:
│       Completed:
│
├── 📄 Nmap
│       [quick scan output]
│       [full scan output]
│
├── 📄 Enumeration
│       Port 80:
│       Port 443:
│       SMB:
│       Other:
│
├── 📄 Foothold
│       Vector:
│       Steps:
│
├── 📄 PrivEsc
│       Enumeration:
│       Vector:
│       Steps:
│
├── 📄 Loot
│   ├── 📄 Credentials
│   │       user : password
│   │       (highlight red)
│   └── 📄 Flags
│           user.txt :
│           root.txt :
│           (highlight red)
│
└── 📄 Rabbit Holes
        [dead ends, wrong paths, time spent — useful for writeups]
```

The Rabbit Holes node is worth keeping. When you write the post-box writeup, the dead ends are part of the story and show your actual thought process. Most writeups skip them. The good ones do not.

---

## PT 5 — Tools by Phase

No tool does everything. This is what I reach for at each phase and why.

### Enumeration

| Tool | Use |
|------|-----|
| `nmap` | Port scanning, service detection, default scripts |
| `rustscan` | Fast initial port discovery, pipe into nmap |
| `gobuster` | Web directory and file fuzzing |
| `ffuf` | Faster web fuzzing, better for parameter and vhost discovery |
| `enum4linux-ng` | SMB/RPC/LDAP enumeration on Windows targets |
| `ldapsearch` | Raw LDAP queries against AD |
| `rpcclient` | RPC enumeration, user and group enumeration against AD |
| `nikto` | Web vulnerability scanning (noisy, run it last) |

Quick note on `rustscan`: it is significantly faster than nmap for initial port discovery. Use it to find open ports fast, then feed the results into nmap for service detection:

```bash
rustscan -a $TARGET -- -sC -sV -oN nmap/full.nmap
```

### Web

| Tool | Use |
|------|-----|
| `burpsuite` | Intercept, modify, replay — the irreplaceable core |
| `ffuf` | Vhost, subdomain, parameter fuzzing |
| `gobuster` | Directory and file fuzzing |
| `whatweb` | Technology fingerprinting |
| `wfuzz` | Parameter fuzzing, IDOR enumeration |
| `sqlmap` | SQL injection testing and exploitation |

Wordlists: SecLists is the standard. `/usr/share/seclists/` on Kali. If it is not installed: `sudo apt install seclists -y`.

### Exploitation

| Tool | Use |
|------|-----|
| `metasploit` | Known CVEs, staged payloads, post-exploitation modules |
| `searchsploit` | Offline ExploitDB search |
| `msfvenom` | Payload generation (shells, DLLs, MSI files) |
| `impacket` | The AD exploitation suite — psexec, wmiexec, secretsdump, GetNPUsers |
| `evil-winrm` | WinRM shell with pass-the-hash support |

### Post-Exploitation

| Tool | Use |
|------|-----|
| `linpeas` / `winpeas` | Automated privesc enumeration — run early |
| `bloodhound` | AD attack path visualization |
| `sharphound` | BloodHound data collection (run on the Windows target) |
| `mimikatz` | Credential dumping, pass-the-hash, Kerberos ticket abuse |
| `powerview` | PowerShell AD enumeration and exploitation |
| `chisel` | TCP tunneling for pivoting through restricted networks |
| `ligolo-ng` | Creates a real tun interface — cleaner than proxychains |

### Password Cracking

| Tool | Use |
|------|-----|
| `hashcat` | GPU-accelerated cracking — use whenever you have a GPU |
| `john` | CPU cracking, excellent for file formats (zip, pdf, ssh keys) |
| `hydra` | Network service brute force (SSH, FTP, HTTP forms) |
| `cewl` | Generate wordlists from target websites |

Always try `rockyou.txt` first. Then `rockyou.txt` with `best64.rule`. Then custom wordlists with `cewl` against the target domain.

---

## PT 6 — The Methodology

### Step 1: Enumerate Before Everything

Resist the urge to go straight to exploitation. Every box that I have gotten stuck on, the cause was incomplete enumeration. Every single time.

The order:

```bash
# 1. Quick scan — what is open
quickscan

# 2. Full port scan in the background — catch non-standard ports  
fullscan &

# 3. Work the quick results while the full scan runs
```

When the full scan finishes, go back and enumerate anything you missed. A web app on port 8080 that quick scan missed because it only hit common ports is where the foothold lives half the time.

### Step 2: Build the Attack Surface

Go through every open port and enumerate it completely before moving on:

- **HTTP/HTTPS**: Technology stack, directory scan, source code, cookies, JavaScript files, API endpoints, vhosts
- **SMB**: Anonymous access, share listing, interesting files, password policy
- **LDAP/RPC**: User enumeration, group enumeration, naming contexts
- **Other services**: Check searchsploit, check the version specifically, check default credentials

Document everything in CherryTree as you go. Do not trust your memory.

### Step 3: Find One Thread and Pull It

Enumeration gives you a list. The foothold comes from identifying which item on that list is actually exploitable. This requires ranking, not just listing.

Questions to ask about each finding:

- Can I act on this right now, or does it require access I do not have?
- Does this match a known vulnerability pattern?
- What does this tell me about the target that I did not know before?

Pull the most promising thread first. If it goes nowhere after a reasonable amount of effort, document it in the Rabbit Holes node and try the next one. Do not spend an hour on a dead end because you are committed to it.

### Step 4: Get a Stable Shell

A reverse shell straight from the exploit is fragile. Stabilise it immediately:

**Linux:**
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

**Windows:** Upgrade to a proper session — evil-winrm if WinRM is available, psexec if you have admin credentials. A plain netcat session on Windows is difficult to work with.

### Step 5: Enumerate Again From the New Context

Everything changes once you have a foothold. Run linpeas or winpeas immediately. What you could not see as an unauthenticated user is now visible. What the service account can access that anonymous enumeration did not show is now in front of you.

The methodology is recursive. Enumerate → foothold → enumerate → escalate → enumerate.

### Step 6: Document As You Go

Do not wait until the box is finished to write anything down. Document each step immediately — the command, the output, the reasoning. Writeups written from memory after the fact are always worse than notes taken in the moment.

The Loot node gets updated every time you find a credential or a flag. Highlighted red. Always.

---

## The Setup Checklist

```
[ ] Kali VM — 4+ cores, 8GB RAM, 80GB fixed disk
[ ] Terminator installed and configured (~/.config/terminator/config)
[ ] Hack Nerd Font installed
[ ] ~/.bashrc updated and sourced
[ ] ~/notes/ populated with cheatsheets
[ ] bat installed and aliased
[ ] rlwrap installed
[ ] SecLists installed
[ ] CherryTree installed
[ ] Master template created at ~/htb/template.ctb
[ ] HTB VPN downloaded and tested
```

Every box starts the same way: `newbox <name>`, connect VPN, `vpnip`, `htb set <target-ip>`, open CherryTree from template, open Terminator in quad layout. The setup is invisible. The work is all that is left.
