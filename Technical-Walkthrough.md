---
layout: default
title: "GnuPG Key Technical Walkthrough"
---
## GnuPG Key Technical Walkthrough

Create a master key with Certification capability and then 3 subkeys with these capabilities:  Sign, Encrypt ant Authenticate.

Then for increasing the secutity, the master private key must be stored on an encrypted storage device and deleted from the computer.

The subkeys can also be offloaded to a smart card for going an extra-mile in the GPG keys protection.

---

- **[Create gpg key pair](./Technical-Walkthrough.md#create-gpg-key-pair)**
- **[Create Subkeys](./Technical-Walkthrough.md#create-subkeys)**
- **[Key Rotation? Sign this new key ](./Technical-Walkthrough.md#key-rotation-sign-this-new-key)**
- **[Protect your master key](./Technical-Walkthrough.md#protect-your-master-key)**
- **[Use the offline master key](./Technical-Walkthrough.md#use-the-offline-master-key)**
- **[Protect your subkeys](./Technical-Walkthrough.md#protect-your-subkeys)**
- **[Test your key](./Technical-Walkthrough.md#test-your-key)**
- **[Import / Re-import](./Technical-Walkthrough.md#import--re-import)**
- **[Deletion / Revocation](./Technical-Walkthrough.md#deletion--revocation)**
- **[References](./Technical-Walkthrough.md#references)**

---

### Create gpg key pair

create a template:

```bash
cat << EOF > primary_gpg_key_constructor.tpl
%echo Generating a RSA key 8192
Key-Type: RSA
Key-Length: 8192
#Key-usage can be cert,sign,auth
Key-Usage: cert
Name-Real: My Name
#Name-Comment:
Name-Email: something@something
Expire-Date: 0
# Do a commit here, so that we can later print "done" :-)
%commit
%echo done
EOF
```

GnuPG asks for the passphrase through pinentry. Do not place a real passphrase
in the template or command line.



#### generate the key

`gpg --batch --generate-key  --enable-large-rsa primary_gpg_key_constructor.tpl`

(`enable-large-rsa` is to have a key of size over 4096 bits)

#### Check your private keys

`gpg --list-secret-keys`

```
sec   rsa8192 2023-01-14 [C]
      E6B0933DD47E3B2F823334F0A567B3B17C27BF6D
uid           [ultimate] My Name <something@something>
```



#### Check your public keys

`gpg --list-keys`

```
pub   rsa8192 2023-01-14 [C]
      E6B0933DD47E3B2F823334F0A567B3B17C27BF6D
uid           [ultimate] My Name <something@something>
```



### Create Subkeys

Edit your key 

`gpg --expert --edit-key E6B0933DD47E3B2F823334F0A567B3B17C27BF6D`

then add a key 

```
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 
```

##### Signature key

To create a RSA 4096 bits sign key:
Select 4 , set the bit size, the expiration date and validate.

##### Encryption key

To create a RSA 4096 bits encryption key:
Add another key with `addkey`, select 6, set the bit size, the expiration date and validate.

##### Authentication key

To create a RSA 4096 bits authentication key:
Add the last key with `addkey`, select 8 then notice the `Current allowed actions` is already set to `Sign Encrypt` 
Unset these capabilities by selecting `S` and `E`
Then set the authenticate capability by selecting `A`

then `Q` to finish, set the bit size, the expiration date and validate.

Then exit gpg nicely by entering `q` then `y`

#### Check all the Subkeys

` gpg --fingerprint --fingerprint E6B0933DD47E3B2F823334F0A567B3B17C27BF6D`

the `--fingerprint` twice is to reveal the subkeys information
```
pub   rsa8192 2023-01-14 [C]
      E6B0 933D D47E 3B2F 8233  34F0 A567 B3B1 7C27 BF6D
uid           [ultimate] My Name <something@something>
sub   rsa4096 2023-01-14 [S] [expires: 2030-01-12]
      61DF 146E F7F6 0971 5E60  46B8 8C75 87C8 6EAC 2F2F
sub   rsa4096 2023-01-14 [E] [expires: 2030-01-12]
      DBBC AE3F 8166 9FF4 75B7  5F35 51D8 8833 7727 23AD
sub   rsa4096 2023-01-14 [A] [expires: 2030-01-12]
      F4CF F234 E969 2304 CC58  D55B D13A E6B6 5E8B E723
```

### Key Rotation? Sign this new key 

In case it is not your first key and you are doing a key rotation:  you can sign the new master key with your previous key that you're transitioning from.

`gpg --sign-key <FULL_PRIMARY_KEY_FINGERPRINT>`

---

### Protect your master key

In order to protect your master key, you need to remove it from the local computer. Before to do so, you need to **securely store** the master key on an **encrypted** external usb device. 

*In the usb drive, create a folder `secret`. Optionally, set a local versioning with `git init` in this folder.*

(The reason to create a local git is to track change on the `secret` folder )

##### Export your master key
Below are the commands to save the master key:

```bash
#!/usr/bin/env bash
set -euo pipefail
umask 077

ENCRYPTED_MOUNT="/run/media/user/Encrypted_Device"
EXPORT_DIR="$ENCRYPTED_MOUNT/secrets/gpg"
EXPORT_FILENAME="MySelf_Master_MyComment_private_rsa8192"
EXPORT_FINGERPRINT="<FULL_PRIMARY_KEY_FINGERPRINT>"

# Never create the expected mount path on the unencrypted root filesystem.
if ! mountpoint -q -- "$ENCRYPTED_MOUNT"; then
    echo "Encrypted device is not mounted at $ENCRYPTED_MOUNT" >&2
    exit 1
fi
if [[ ! -d "$ENCRYPTED_MOUNT" || ! -w "$ENCRYPTED_MOUNT" ]]; then
    echo "Encrypted mount is not writable" >&2
    exit 1
fi

# This is safe only after the mount check above.
install -d -m 0700 -- "$EXPORT_DIR"

ARTIFACTS=(
    "$EXPORT_DIR/$EXPORT_FILENAME.private-key.sec"
    "$EXPORT_DIR/$EXPORT_FILENAME.subkeys.sec"
    "$EXPORT_DIR/$EXPORT_FILENAME.pubkey.asc"
    "$EXPORT_DIR/$EXPORT_FILENAME.keygrip.txt"
    "$EXPORT_DIR/gnupg-ownertrust.txt"
    "$EXPORT_DIR/revoke.no-reason.asc"
    "$EXPORT_DIR/revoke.compromised.asc"
    "$EXPORT_DIR/revoke.superseded.asc"
    "$EXPORT_DIR/revoke.no-longer-used.asc"
)
for ARTIFACT in "${ARTIFACTS[@]}"; do
    [[ ! -e "$ARTIFACT" ]] || {
        echo "Refusing to overwrite $ARTIFACT" >&2
        exit 1
    }
done

gpg --armor --output "$EXPORT_DIR/$EXPORT_FILENAME.private-key.sec" \
    --export-secret-keys "$EXPORT_FINGERPRINT"
gpg --armor --output "$EXPORT_DIR/$EXPORT_FILENAME.subkeys.sec" \
    --export-secret-subkeys "$EXPORT_FINGERPRINT"
gpg --armor --output "$EXPORT_DIR/$EXPORT_FILENAME.pubkey.asc" \
    --export "$EXPORT_FINGERPRINT"
gpg --list-secret-keys --fingerprint --with-keygrip "$EXPORT_FINGERPRINT" \
    > "$EXPORT_DIR/$EXPORT_FILENAME.keygrip.txt"
gpg --export-ownertrust > "$EXPORT_DIR/gnupg-ownertrust.txt"

declare -A REVOCATION_REASONS=(
    [no-reason]=0
    [compromised]=1
    [superseded]=2
    [no-longer-used]=3
)
for REASON in no-reason compromised superseded no-longer-used; do
    OUTPUT="$EXPORT_DIR/revoke.$REASON.asc"
    printf '%s\n' y "${REVOCATION_REASONS[$REASON]}" "" y |
        gpg --no-tty --command-fd 0 --armor --output "$OUTPUT" \
            --gen-revoke "$EXPORT_FINGERPRINT"
done

chmod 0600 -- "${ARTIFACTS[@]}"
```

Use the **full primary fingerprint**, not a short or long key ID. A fingerprint
identifies an OpenPGP key; a keygrip is a different identifier used by
`gpg-agent` for one secret-key component. `mountpoint` confirms that something
is mounted at the expected path; you must still confirm that it is your unlocked
encrypted device.

The files have these purposes:

- `private-key.sec`: complete primary-key and subkey disaster recovery.
- `subkeys.sec`: operational recovery containing secret subkeys only.
- `pubkey.asc`: convenient standalone public-key export.
- `keygrip.txt`: diagnostic and reference information.
- `gnupg-ownertrust.txt`: the local GnuPG ownertrust database.

The old `pgp-ownertrust.asc` name contained the same plain-text data produced by
`gpg --export-ownertrust`; it was not an ASCII-armored OpenPGP object. Restore
the renamed file with:

```bash
gpg --import-ownertrust "$EXPORT_DIR/gnupg-ownertrust.txt"
```

*I prefer to generate all possible revocation certificates just in case one is
needed one day.* The automated selection is:

```text
0 = no reason specified
1 = key compromised
2 = key superseded
3 = key no longer used
```

Pinentry still asks for the primary-key passphrase securely. Never put that
passphrase in the script, environment, command line, or shell history. Any one
of these certificates can revoke the primary key; keeping four only lets you
choose the appropriate reason later. Protect them with the private backups, and
never silently overwrite a known-good backup.

GnuPG normally creates `openpgp-revocs.d/<fingerprint>.rev` when it generates a
primary key. Importing a secret key into a new home does not necessarily recreate
that file. Do not depend on it during a RAM restore: the explicit certificates
above are the persistent copies. Also, `$HOME/.gnupg/openpgp-revocs.d/` is wrong
whenever another `GNUPGHOME` is active. Check the current home with:

```bash
gpgconf --list-dirs homedir
```

Before deleting anything, test the complete backup in an isolated RAM keyring:

```bash
set -euo pipefail
umask 077

EXPORT_DIR="/run/media/user/Encrypted_Device/secrets/gpg"
EXPORT_FILENAME="MySelf_Master_MyComment_private_rsa8192"
EXPORT_FINGERPRINT="<FULL_PRIMARY_KEY_FINGERPRINT>"
TEST_GNUPGHOME="$(mktemp -d /dev/shm/gnupg-test.XXXXXX)"
chmod 0700 "$TEST_GNUPGHOME"

cleanup_test_gnupghome() {
    if [[ -d "$TEST_GNUPGHOME" && ! -L "$TEST_GNUPGHOME" &&
          "$TEST_GNUPGHOME" == /dev/shm/gnupg-test.* ]]; then
        gpgconf --homedir "$TEST_GNUPGHOME" --kill all || true
        rm -rf -- "$TEST_GNUPGHOME"
        unset TEST_GNUPGHOME
    else
        echo "Refusing to delete suspicious TEST_GNUPGHOME" >&2
        return 1
    fi
}
trap cleanup_test_gnupghome EXIT

gpg --homedir "$TEST_GNUPGHOME" \
    --import "$EXPORT_DIR/$EXPORT_FILENAME.private-key.sec"
SECRET_LIST="$(gpg --homedir "$TEST_GNUPGHOME" \
    --list-secret-keys --fingerprint --with-keygrip "$EXPORT_FINGERPRINT")"
printf '%s\n' "$SECRET_LIST"

PRIMARY_STATUS="$(awk '/^sec/{print $1; exit}' <<< "$SECRET_LIST")"
SUBKEY_COUNT="$(awk '$1 == "ssb" {count++} END {print count + 0}' \
    <<< "$SECRET_LIST")"
VALID=0
[[ "$PRIMARY_STATUS" == sec && "$SUBKEY_COUNT" -ge 3 ]] || VALID=1

cleanup_test_gnupghome
trap - EXIT

(( VALID == 0 )) || {
    echo "Backup test failed: expected sec and at least three usable ssb entries" >&2
    exit 1
}
```

The test keyring must show `sec`, not `sec#`, and actual `ssb` entries for the
S/E/A subkeys. A successful export command by itself is not sufficient: exports
made after keys became unavailable or were diverted to a smartcard can contain
stubs. For diagnosis, `gpg --list-packets` distinguishes actual protected secret
material from `gnu-dummy S2K` and `gnu-divert-to-card S2K` representations.

##### Master key deletion

To increase the security, it is advised to remove the main key, and to only use the subkeys.

Only continue after the independent backup test above succeeds. The preferred
modern command deletes only the primary secret component:

```bash
gpg --delete-secret-keys "${EXPORT_FINGERPRINT}!"
gpg --list-secret-keys "$EXPORT_FINGERPRINT"
```

The quoted `!` forces selection of exactly the primary key. Without it, deleting
the primary secret key can also delete its secret subkeys.

For the lower-level `gpg-connect-agent` alternative, first list the keygrips:

```
gpg --list-secret-keys --with-keygrip "$EXPORT_FINGERPRINT"
<active GNUPGHOME>/pubring.kbx
------------------------
sec   rsa8192 2023-01-14 [C]
      E6B0933DD47E3B2F823334F0A567B3B17C27BF6D
      Keygrip = A40B888D1142C025A373D11876449440F29A7733
uid           [ultimate] My Name <something@something>
ssb   rsa4096 2023-01-14 [S] [expires: 2030-01-12]
      Keygrip = C0D4DCC01E66CE529BFB57EC315897B250A8CD84
ssb   rsa4096 2023-01-14 [E] [expires: 2030-01-12]
      Keygrip = B813802B83E916DC11609F6C02B2AA675B175295
ssb   rsa4096 2023-01-14 [A] [expires: 2030-01-12]
      Keygrip = DFDCE0D68B5CC70FED754A4148505FEF82E7189D

```



`DELETE_KEY` expects the keygrip printed under the primary `sec` entry. It does
**not** accept a GPG key ID, long key ID, or OpenPGP fingerprint:

```
gpg-connect-agent "DELETE_KEY <PRIMARY_SEC_KEYGRIP>" /bye
```



Ensure that the private master key has been removed

```
gpg --list-secret-keys "$EXPORT_FINGERPRINT"
<active GNUPGHOME>/pubring.kbx
------------------------------
sec#  rsa8192 2023-01-14 [C]
      E6B0933DD47E3B2F823334F0A567B3B17C27BF6D
uid           [ultimate] My Name <something@something>
ssb   rsa4096 2023-01-14 [S] [expires: 2030-01-12]
ssb   rsa4096 2023-01-14 [E] [expires: 2030-01-12]
ssb   rsa4096 2023-01-14 [A] [expires: 2030-01-12]
```



You should see the `#` symbol after `sec`, confirming that the private primary
key is not usable. If the subkeys are on a YubiKey, the expected result is
`sec#` followed by three `ssb>` entries instead of the local `ssb` entries shown
above.


### Use the offline master key

Use a RAM-backed temporary `GNUPGHOME`, not `~/gpgtmp`. Keep it isolated from
the normal workstation keyring; in particular, do not point `--keyring` at the
normal `pubring.kbx`.

```bash
ENCRYPTED_MOUNT="/run/media/user/Encrypted_Device"
EXPORT_DIR="$ENCRYPTED_MOUNT/secrets/gpg"
EXPORT_FILENAME="MySelf_Master_MyComment_private_rsa8192"
EXPORT_FINGERPRINT="<FULL_PRIMARY_KEY_FINGERPRINT>"

mountpoint -q -- "$ENCRYPTED_MOUNT" || {
    echo "Encrypted device is not mounted" >&2
    exit 1
}

ORIGINAL_GNUPGHOME_SET="${GNUPGHOME+x}"
ORIGINAL_GNUPGHOME="${GNUPGHOME-}"
NORMAL_GNUPGHOME="$(gpgconf --list-dirs homedir)"

GNUPGHOME="$(mktemp -d /dev/shm/gnupg.XXXXXX)"
export GNUPGHOME
chmod 0700 "$GNUPGHOME"

gpg --import "$EXPORT_DIR/$EXPORT_FILENAME.private-key.sec"
gpg --import-ownertrust "$EXPORT_DIR/gnupg-ownertrust.txt"
gpg --list-secret-keys --fingerprint --with-keygrip "$EXPORT_FINGERPRINT"
```

The indicators have precise meanings:

```text
sec    actual primary secret key available
sec#   primary secret key unavailable/offline
ssb    actual secret subkey locally available
ssb#   secret subkey unavailable
ssb>   secret subkey located on a smartcard/YubiKey
```

This restored keyring must show `sec`, not `sec#`. A complete backup should also
show actual `ssb` entries for the S/E/A subkeys. The imported backup contains its
own public certificate, so sharing the normal workstation's `pubring.kbx` is
neither necessary nor desirable.

```bash
gpg --edit-key "$EXPORT_FINGERPRINT"
```

You can now perform any operations you need to do with your master key

Changes made here do not modify the encrypted `.private-key.sec` backup. After
changing a passphrase, expiration, subkey, identity, or certification, create a
new candidate without overwriting the known-good backup:

```bash
NEW_BACKUP="$EXPORT_DIR/$EXPORT_FILENAME.private-key.NEW.sec"
[[ ! -e "$NEW_BACKUP" ]] || {
    echo "Refusing to overwrite $NEW_BACKUP" >&2
    exit 1
}
gpg --armor --output "$NEW_BACKUP" \
    --export-secret-keys "$EXPORT_FINGERPRINT"
chmod 0600 "$NEW_BACKUP"
```

Test the candidate in another isolated RAM keyring:

```bash
NEW_TEST_GNUPGHOME="$(mktemp -d /dev/shm/gnupg-test.XXXXXX)"
chmod 0700 "$NEW_TEST_GNUPGHOME"
gpg --homedir "$NEW_TEST_GNUPGHOME" --import "$NEW_BACKUP"
NEW_SECRET_LIST="$(gpg --homedir "$NEW_TEST_GNUPGHOME" \
    --list-secret-keys --fingerprint --with-keygrip "$EXPORT_FINGERPRINT")"
printf '%s\n' "$NEW_SECRET_LIST"

NEW_PRIMARY_STATUS="$(awk '/^sec/{print $1; exit}' <<< "$NEW_SECRET_LIST")"
NEW_SUBKEY_COUNT="$(awk '$1 == "ssb" {count++} END {print count + 0}' \
    <<< "$NEW_SECRET_LIST")"

if [[ -d "$NEW_TEST_GNUPGHOME" && ! -L "$NEW_TEST_GNUPGHOME" &&
      "$NEW_TEST_GNUPGHOME" == /dev/shm/gnupg-test.* ]]; then
    gpgconf --homedir "$NEW_TEST_GNUPGHOME" --kill all || true
    rm -rf -- "$NEW_TEST_GNUPGHOME"
    unset NEW_TEST_GNUPGHOME
else
    echo "Refusing to delete suspicious NEW_TEST_GNUPGHOME" >&2
    exit 1
fi

[[ "$NEW_PRIMARY_STATUS" == sec && "$NEW_SUBKEY_COUNT" -ge 3 ]] || {
    echo "New backup failed validation; keep the known-good backup" >&2
    exit 1
}
```

Only after that test succeeds, preserve the old copy and atomically promote the
candidate on the same encrypted filesystem:

```bash
KNOWN_GOOD="$EXPORT_DIR/$EXPORT_FILENAME.private-key.sec"
ARCHIVE="$KNOWN_GOOD.previous.$(date -u +%Y%m%dT%H%M%SZ)"
[[ -f "$KNOWN_GOOD" && ! -e "$ARCHIVE" ]] || exit 1
cp --preserve=mode,timestamps -- "$KNOWN_GOOD" "$ARCHIVE"
mv -- "$NEW_BACKUP" "$KNOWN_GOOD"
chmod 0600 -- "$KNOWN_GOOD" "$ARCHIVE"
```

Keep the archived backup. If public identities or subkeys changed, refresh the
public and subkeys-only exports using the same new-file, validate, and preserve
approach.

Then kill the agent and remove the temporary home to set the master key offline
again. The path guard is mandatory:

```bash
if [[ -n "${GNUPGHOME:-}" && -d "$GNUPGHOME" && ! -L "$GNUPGHOME" &&
      "$GNUPGHOME" == /dev/shm/gnupg.* ]]; then
    gpgconf --homedir "$GNUPGHOME" --kill all || true
    rm -rf -- "$GNUPGHOME"
    unset GNUPGHOME
else
    echo "Refusing to delete suspicious GNUPGHOME" >&2
    exit 1
fi

if [[ "$ORIGINAL_GNUPGHOME_SET" == x ]]; then
    export GNUPGHOME="$ORIGINAL_GNUPGHOME"
fi
CURRENT_GNUPGHOME="$(gpgconf --list-dirs homedir)"
printf '%s\n' "$CURRENT_GNUPGHOME"
[[ "$CURRENT_GNUPGHOME" == "$NORMAL_GNUPGHOME" ]] || {
    echo "The original GnuPG home was not restored" >&2
    exit 1
}
```

The last command must show the original normal GnuPG home. Verify once more in
that normal home that the primary is `sec#`; YubiKey-backed subkeys appear as
`ssb>`.

```text
Encrypted offline storage
├── complete primary + subkeys secret backup
├── subkeys-only backup, public key, revocations, keygrips, ownertrust
│          │ temporary restore
│          ▼
│   RAM-backed GNUPGHOME: sec [C], ssb [S/E/A]
│          │ maintenance, re-export, independent validation
│          ▼
└── destroy RAM GNUPGHOME

Normal workstation: sec# [C], with operational ssb or YubiKey-backed ssb>
```

---

### Protect your subkeys

You don't want any of your keys on your computer ? Dear paranoid friend, I hear you!
OpenPGP smartcards is one good approach.

Here are the steps for a Yubikey smartcard:

Before moving any subkey, follow the dedicated [YubiKey Setup](./YubiKey-Setup.md)
page. It covers inspection, OpenPGP and PIV credentials, retry counters, touch
policies, and firmware compatibility without resetting the device.

Once this setup is complete, move your subkeys to the smartcard:

````
gpg --edit-key E6B0933DD47E3B2F823334F0A567B3B17C27BF6D
...
gpg> key 1
gpg> keytocard
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]

Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1
````

remember to **unselect** your current subkey before selecting a new one by entering `key 1` again

Perform the same operation to move the other keys to your smartcard.
then `save	`

Writing a subkey can reset that slot's touch policy. Reapply and verify the
policies after all three `keytocard` operations:

```bash
ykman openpgp keys set-touch sig on
ykman openpgp keys set-touch dec on
ykman openpgp keys set-touch aut on
ykman openpgp info
```

Here is what the result should be:

```
gpg --card-status  
Reader ...........: 0000:0000:A:0
Application ID ...: 0000000000000000000000000000
Application type .: OpenPGP
Version ..........: 2.0
Manufacturer .....: Yubico
Serial number ....: 00000000
Name of cardholder: My Name
Language prefs ...: en
Salutation .......: 
URL of public key : <Whatever PGP server on which you uploaded your key>
Login data .......: Something
Signature PIN ....: forced
Key attributes ...: rsa4096 rsa4096 rsa4096
Max. PIN lengths .: 127 127 127
PIN retry counter : 12 3 6
Signature counter : 144
Signature key ....: 61DF 146E F7F6 0971 5E60  46B8 8C75 87C8 6EAC 2F2F
      created ....: 2020-01-12 10:09:27
Encryption key....: DBBC AE3F 8166 9FF4 75B7  5F35 51D8 8833 7727 23AD
      created ....: 2020-01-12 10:06:54
Authentication key: F4CF F234 E969 2304 CC58  D55B D13A E6B6 5E8B E723
      created ....: 2020-01-12 10:09:58
General key info..: sub  rsa4096/8C7587C86EAC2F2F 2020-01-12 My Name <something@something>
sec#  rsa8192/A567B3B17C27BF6D  created: 2020-01-12  expires: never     
ssb>  rsa4096/51D88833772723AD  created: 2020-01-12  expires: 2030-01-12
                                card-no: 0000 00000000
ssb>  rsa4096/8C7587C86EAC2F2F  created: 2020-01-12  expires: 2030-01-12
                                card-no: 0000 00000000
ssb>  rsa4096/D13AE6B65E8BE723  created: 2020-01-12  expires: 2030-01-12
                                card-no: 0000 00000000

```



#### Yubitouch

To go further in your setup I suggest checking this [YubiTouch repository](https://github.com/a-dma/yubitouch/)


---

### Test your key

##### Encrypt a file

```
gpg --encrypt --sign --armor -r receipient@gmail.com message.txt
```

The `-encrypt` option tells gpg to encrypt the file, and the `--sign` option tells it to sign the file with your details. The `--armor` option tells gpg to create an ASCII file. The `-r` (recipient) option must be followed by the email address of the person you’re sending the file to.

The `message.txt` is the name of the file.

The file is created with the same name as the original, but with “.asc” appended to the file name (ex. `message.asc`)

##### Decrypt a file

`gpg --decrypt coded.asc`

The `coded.asc` is the file received. 

To redirect the output into another file :

`gpg --decrypt coded.asc > plain.txt`

---

### Import / Re-import

#### Import someone's public key:

```
gpg --import pgp-public-keys.asc
```

The `pgp-public-keys.asc` is the key name of that person.

#### Reimport your own keys

in case you ever need to reimport your keys:

```bash
EXPORT_DIR="/run/media/user/Encrypted_Device/secrets/gpg"
EXPORT_FILENAME="MySelf_Master_MyComment_private_rsa8192"
EXPORT_FINGERPRINT="<FULL_PRIMARY_KEY_FINGERPRINT>"

gpg --import "$EXPORT_DIR/$EXPORT_FILENAME.pubkey.asc"
gpg --import "$EXPORT_DIR/$EXPORT_FILENAME.subkeys.sec"
gpg --import-ownertrust "$EXPORT_DIR/gnupg-ownertrust.txt"
gpg --list-secret-keys "$EXPORT_FINGERPRINT"
```

This is the normal workstation import: the primary should remain `sec#`, while
the operational subkeys appear as `ssb` locally or `ssb>` on a YubiKey. Import
the complete `private-key.sec` only for disaster recovery or into the isolated
RAM `GNUPGHOME` described in [Use the offline master key](#use-the-offline-master-key).
Importing it into the normal home deliberately brings the primary secret key
back online.

---

### Deletion / Revocation

#### You messed up? Delete / Revoke your subkey

Something went wrong at the creation, you can delete a subkey? 
Edit your key 

`gpg --expert --edit-key E6B0933DD47E3B2F823334F0A567B3B17C27BF6D`

select the `subkey` or `user ID` (depending on what went wrong)

`key F2DF28C8CDF28C`

Then delete the key

`deluid`

**If you dont select the key... well you'll loose all of them**

*Same principle applies for `uid` and `deluid`*

 #### Deletion of a main key 

##### secret key

```bash
gpg --list-secret-keys --fingerprint "<FULL_PRIMARY_KEY_FINGERPRINT>"
gpg --delete-secret-keys "<FULL_PRIMARY_KEY_FINGERPRINT>"
```

(This will also delete all subkeys associated with this secret key)

##### public key

```bash
gpg --fingerprint "<FULL_PRIMARY_KEY_FINGERPRINT>"
gpg --delete-keys "<FULL_PRIMARY_KEY_FINGERPRINT>"
```

#### Revocation

One of your subkey has been compromised, let everyone know!

**Select the subkey!**

`key F2DF28C8CDF28C`

Then delete the key

`revkey`





### References

https://incenp.org/notes/2015/using-an-offline-gnupg-master-key.html

https://gist.github.com/ageis/5b095b50b9ae6b0aa9bf

https://riseup.net/en/security/message-security/openpgp/gpg-best-practices

https://www.devdungeon.com/content/gpg-tutorial

https://8gwifi.org/docs/gpg.jsp

Yubikey : https://www.youtube.com/watch?v=xGsixSh6sC4
