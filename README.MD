# Week 3 - Password Cracking

NetworkWalks Internship, Batch B082 - Week 3

One module:

| Module | Tools | Type |
|---|---|---|
| PM1 - Password Cracking | John the Ripper + Johnny GUI | Offline |

Week 2 was about seeing what is on a network. This week is about getting past a
credential once you have the encrypted data in hand. The target is a
password-protected PDF, and the whole exercise is offline: nothing is sent to the
file's owner, no login is attempted anywhere. The work is done against a hash
extracted from the file, on my own machine.

---

# Module PM1 - Cracking a PDF password

## What this is

Recovering the open password of an encrypted PDF (`My Locked PDF1.pdf`) on a Windows
host, using John the Ripper as the cracking engine and Johnny as the GUI front end.

The PDF is not attacked directly. Instead the encryption parameters are pulled out of
the file into a hash string, and John runs against that string offline until it finds
a password that reproduces it. This is the same reason theHarvester never touched
Microsoft in Week 2 - the actual work happens on data, not on the live target.

## Background

A password-protected PDF does not store the password. It stores a value derived from
the password through a key-derivation function, plus the parameters needed to check a
guess. To crack it you extract those parameters into a hash and let a cracker try
candidate passwords, running each one through the same derivation and comparing the
result. When they match, the password is found.

Two pieces do the job:

- **John the Ripper** is the engine. The jumbo build ships with `pdf2john`, which
  reads a PDF and prints its hash, and it knows the `PDF` hash format so it can crack
  what `pdf2john` produces. Core John does not - the jumbo edition is what adds the
  extra formats and attack modes.
- **Johnny** is a GUI wrapper around John. It does no cracking itself; it calls the
  `john.exe` you point it at and shows the results in a table. Everything it does can
  be done from the command line, but the GUI makes the load / attack / read-result
  loop visible.

Why it matters: encryption on a document is only as strong as the password behind it.
A short or guessable password on an otherwise strong cipher falls in seconds, because
the attacker is not breaking the maths - they are guessing the one weak input to it.

## Environment

Windows 11 host. Everything runs natively on Windows this week - no VM, because John's
Windows jumbo build is self-contained.

- **John the Ripper 1.9.0-jumbo-1 OMP** `[cygwin 64-bit x86_64 AVX2 AC]`
- **Johnny** GUI front end
- **Online HashCrack** PDF Hash Extractor (browser) to run `pdf2john` without a Kali VM

John's jumbo build is a cygwin package, which is why Johnny reports it as a cygwin
64-bit binary even though it is running on Windows. The `AVX2` in the banner is the
CPU instruction set John detected and will use to speed up the guessing.

## Scope

The PDF is mine and I set its password myself, so there is no question of consent.
Offline password cracking is only legitimate against data you own or are authorised to
test. The technique is identical whether the file is yours or someone else's - the
difference is entirely permission, and that is the line that keeps this a lab exercise
rather than an offence.

## Step 1 - Install John the Ripper

Downloaded the Windows jumbo build of John the Ripper from the official site and
extracted it. The important file inside the archive is `run/john.exe` - that is the
executable Johnny needs to be pointed at:

```
C:/Users/ghazi/Downloads/john-1.9.0-jumbo-1-win64/john-1.9.0-jumbo-1-win64/run/john.exe
```

Nothing to compile or configure - the jumbo build runs as extracted.

## Step 2 - Install Johnny and point it at John

Downloaded Johnny from the official Openwall page and ran it. On first launch the
Passwords table is empty and no attack can be started yet, because Johnny has not been
told where John is.

![Johnny on first launch, empty and unconfigured](screenshots/johny_run.png)

Johnny needs one thing before it can do anything: the path to `john.exe`. Under
**Settings**, set **John the Ripper executable** to the `run/john.exe` from Step 1 and
Johnny confirms it read the binary:

```
Detected John the Ripper 1.9.0-jumbo-1 OMP [cygwin 64-bit x86_64 AVX2 AC]
```

![Settings tab with john.exe mapped and detected](screenshots/John_Settings_MAP.png)

That "Detected ..." line is the check that matters. Until it appears, Johnny is a
window with nothing behind it - it does no cracking of its own, so the whole tool is
useless until this path is correct. The banner it echoes back is John identifying its
own version and CPU features, which is how I know the jumbo build (the one with PDF
support) is the one wired in, not core John.

## Step 3 - Extract the hash from the PDF

John's jumbo build includes `pdf2john`, but rather than run it from the command line I
used the **Online HashCrack** PDF Hash Extractor, which runs the same `pdf2john` in the
browser. Uploaded `My Locked PDF1.pdf`:

![Online HashCrack PDF hash extractor with the locked PDF uploaded](screenshots/Locked_PDF.png)

The page states plainly what it is: *"This site is using pdf2john from JohnTheRipper to
extract the hash"* and *"Uploaded files will be deleted immediately. We do NOT store
your files."* So this is a convenience wrapper around the exact tool John ships with -
useful for skipping the command line, at the cost of handing a file to a third-party
site.

> **Worth being honest about this trade-off.** Uploading even a file I own to an
> external service is a habit worth questioning. The safe version is to run
> `pdf2john.pl My\ Locked\ PDF1.pdf` locally from John's `run/` folder, which never
> lets the file leave the machine. I used the site because it was quick, but for
> anything that is not a throwaway lab PDF I would extract the hash locally.

The extractor returned the hash, which I saved to [`Hashed.txt`](Hashed.txt):

```
$pdf$4*4*128*-1028*1*16*ca7f72f11459cba469f1005a8765ed51*32*f32d8fa1bfbe2648226dffc39f7909ea0021446990b9e4114071a4d9104984c1*32*9322f50c29569712067a775264635e4954ccb1b99e209d664984054ffad30a6a
```

Reading the fields tells you what you are up against before cracking anything:

| Field | Value | Meaning |
|---|---|---|
| `$pdf$` | - | PDF hash format marker |
| `4*4` | 4, 4 | encryption revision / version |
| `128` | 128 | key length in bits |
| `-1028` | -1028 | permissions flags |

The `$pdf$4*4*128*...` shape matches **PDF 1.4 - 1.6 (Acrobat 5 - 8)**, which the
extractor's own examples list as hashcat mode 10500. Knowing the version matters
because the derivation cost differs between PDF revisions - later revisions iterate the
hashing far more, so the same password would take much longer to crack on a newer file.

## Step 4 - Load the hash into Johnny

Back in Johnny, the hash is loaded through **Open password file**. The menu offers two
options:

![Open password file menu showing the two format options](screenshots/OPEN_HASH_FILE.png)

| Option | Use it for |
|---|---|
| Open password file (PASSWD format) | Unix `/etc/shadow`-style files |
| **Open other file format (*2john)** | anything produced by a `*2john` tool - including `pdf2john` |

The hash came from `pdf2john`, so the second option is the right one. Picking PASSWD
format here would fail, because the file is not a shadow file - it is `*2john` output.
Browsed to `Hashed.txt` and opened it.

## Step 5 - Start the attack

With the hash loaded, **Start new attack** kicks off John:

![Start new attack, format detected as PDF, 1 hash left to crack](screenshots/start_new_attack_johnny.png)

The status bar tells the whole story before it even finishes:

```
0% (0/1: 0 cracked, 1 left) [format=PDF]
```

Two things stand out. Johnny **auto-detected `format=PDF`** from the hash itself - I did
not have to tell it the type, because the `$pdf$` marker made it unambiguous. And
`1 left` confirms it loaded exactly one hash to work on. Started with John's default
attack (no wordlist configured), which begins with its built-in single-crack and
incremental modes.

## Step 6 - Result

A few seconds later the attack completed:

![Johnny showing the cracked password good-luck at 100%](screenshots/Password_cracked_johnny.png)

```
100% (1/1: 1 cracked, 0 left) [format=PDF]
```

| Column | Value |
|---|---|
| Password | **`good-luck`** |
| Hash | `$pdf$4*4*128*-1028*1*16...` |
| Formats | PDF |

The password is **`good-luck`**. It fell in seconds because it is a short lowercase
dictionary-style phrase - exactly the kind of input John's default modes find fastest.
The 128-bit key length in the hash is irrelevant to how quickly this cracked; the key
is derived from the password, so a weak password means a weak key no matter how many
bits the cipher nominally uses.

## Step 7 - Verify against the real PDF

The proof is opening the actual file. Entering `good-luck` at the PDF's password prompt
opens it with no error:

![PDF password prompt accepting the cracked password](screenshots/password_cracked_PDF.png)

```
'My Locked PDF1.pdf' is protected. Please enter a Document Open Password.
```

The cracked password opened it first try. This step matters - a cracker reporting
"cracked" is a claim about a hash, and opening the real document is what turns that
claim into a confirmed result.

## Problems I ran into

**Johnny did nothing until John was mapped.** On first launch every attack button was
greyed out and there was no obvious error saying why. The tool depends entirely on an
external `john.exe`, and until the Settings path is set and the "Detected ..." line
appears, Johnny cannot do anything. The empty state looked broken when it was really
just unconfigured.

**Almost picked the wrong load option.** "Open password file (PASSWD format)" is the
first and more obvious menu entry, but the hash came from `pdf2john`, which needs
"Open other file format (*2john)". Loading it as PASSWD would have failed to parse.

**Uploaded a file to a third-party site.** The convenient route to the hash was an
online extractor, which meant handing over the PDF. Fine for a lab file I own, but a
habit I would not carry to real work - `pdf2john` runs locally and never needs the file
to leave the machine.

## To do

- Extract the next hash with `pdf2john.pl` locally instead of the web extractor, and
  confirm the output matches.
- Run a real wordlist attack (`rockyou.txt`) and time how a dictionary word falls
  versus John's default modes.
- Try a stronger password (long, mixed character set) on a fresh PDF and watch the
  same attack fail to finish - the point being to feel where password strength actually
  starts to matter.
- Repeat with a newer PDF revision (Acrobat 9+ / hashcat 10600) to see the higher
  iteration count slow the crack down.

## What I took from this

The password is the weak point, not the cipher. The hash advertised a 128-bit key, and
none of that mattered - `good-luck` is a short guessable phrase and John found it in
seconds. Strong encryption around a weak password is a strong lock on an unlocked door.

The GUI is only a front end. Johnny does no cracking; it shells out to John. Everything
it showed - the detected format, the progress, the result - is John's work displayed in
a table. Knowing that is what made the empty first-launch state make sense.

The tool read the format for me. `format=PDF` came straight from the `$pdf$` marker in
the hash, which is why extracting a clean, correctly-formatted hash is half the job. Get
the hash right and the cracker often needs no configuration at all.

Verify against the real thing. "1 cracked" is a statement about a hash. Opening
`My Locked PDF1.pdf` with `good-luck` is what makes it a fact.

---

## A note on use

Everything this week was offline and against a file I own and password-protected
myself. No system was logged into, no live target received any traffic, and the only
data that left my machine was a PDF I control, handed to a hash extractor.

Offline password cracking is a legitimate technique - defenders use exactly this to
audit their own password policies and find weak credentials before an attacker does.
The technique does not change with permission; only the authorisation does. Run it
against your own data, or data you are explicitly authorised to test, and nothing else.

## Author

Ghazi Smach
