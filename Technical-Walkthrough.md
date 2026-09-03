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
Passphrase:  Something
# Do a commit here, so that we can later print "done" :-)
%commit
%echo done
EOF
```



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

`gpg --sign-key <longid>`

---

### Protect your master key

In order to protect your master key, you need to remove it from the local computer. Before to do so, you need to **securely store** the master key on an **encrypted** external usb device. 

*In the usb drive, create a folder `secret`. Optionally, set a local versioning with `git init` in this folder.*

(The reason to create a local git is to track change on the `secret` folder )

##### Export your master key
Below are the commands to save the master key:

```bash
# Change the values of these variables if needed
export EXPORT_DIR="<Encrypted_Device_Path>/secrets/gpg"
export EXPORT_FILENAME="MySelf_Master_MyComment_private_rsa8192"
export EXPORT_KEY_ID="<GPG_Key_ID>"
mkdir -p ${EXPORT_DIR}
gpg --armor --output $EXPORT_DIR/$EXPORT_FILENAME.private-key.sec --export-secret-keys $EXPORT_KEY_ID
gpg --armor --output $EXPORT_DIR/$EXPORT_FILENAME.subkeys.sec --export-secret-subkeys $EXPORT_KEY_ID
gpg --armor --output $EXPORT_DIR/$EXPORT_FILENAME.pubkey.asc --export $EXPORT_KEY_ID
for REASON in no-reasons compromized superseded no-longer-used; do
echo -e "Select the revocation reason: \e[92m$REASON\e[0m"
gpg --output $EXPORT_DIR/$EXPORT_FILENAME.revoke.$REASON.asc --gen-revoke $EXPORT_KEY_ID
done
gpg --list-secret-keys --with-keygrip $EXPORT_KEY_ID > $EXPORT_FILENAME.keygrip.txt
gpg --export-ownertrust > $EXPORT_DIR/pgp-ownertrust.asc
```

*I prefer to generate all possible revocation keys just in case they are needed one day*
Select 0 (No reason specified)
Select 1 (Key has been compromised)
Select 2 (key is superseded)
Select 3 (key is no longer used)

##### Master key deletion

To increase the security, it is advised to remove the main key, and to only use the subkeys.

list the keygrip

```
gpg --list-secret-keys --with-keygrip
/home/user/.gnupg/pubring.kbx
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



Then you can remove the Master private key (specifying the keygrip id to delete)

```
gpg-connect-agent "DELETE_KEY A40B888D1142C025A373D11876449440F29A7733" /bye
```



Ensure that the private master key has been removed

```
gpg2 --list-secret-keys
/home/user/.gnupg/pubring.kbx
------------------------------
sec#  rsa8192 2023-01-14 [C]
      E6B0933DD47E3B2F823334F0A567B3B17C27BF6D
uid           [ultimate] My Name <something@something>
ssb   rsa4096 2023-01-14 [S] [expires: 2030-01-12]
ssb   rsa4096 2023-01-14 [E] [expires: 2030-01-12]
ssb   rsa4096 2023-01-14 [A] [expires: 2030-01-12]
```



you should see the `#` symbol after the sec keyword, confirming that the private master key is not usable.


### Use the offline master key



```
mkdir -p ~/gpgtmp
chmod 0700 ~/gpgtmp
gpg --homedir ~/gpgtmp --import <Encrypted_Device_Path>/secrets/gpg/MySelf_Master_MyComment_private_rsa8192.private-key.sec
gpg --homedir ~/gpgtmp --keyring ~/.gnupg/pubring.kbx --edit-key something@something
```

You can now perform any operations you need to do with your master key

Then kill the agent to set this master key offline again.

```
gpg-connect-agent --homedir ~/gpgtmp KILLAGENT /bye
rm -rf ~/gpgtmp
```

---

### Protect your subkeys

You don't want any of your keys on your computer ? Dear paranoid friend, I hear you! 
OpenPGP smartcards is one good approach.

Here are the steps for a Yubikey smartcard:

**Change the PINs on your Yubikey.** The default PIN of your Yubikey is  `12345678`.

```
$ gpg2 --card-edit
gpg/card> admin
Admin commands are allowed

gpg/card> passwd
1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection?
```

Then you can start to setup the informations of your YubiKey.
the `help` command is your friend, but here are the things I would set:

- `login`
- `lang`
- `url`

Then move your subkeys to the smartcard:

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
PIN retry counter : 3 3 3
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

```
gpg --import pgp-public-keys.asc
gpg --import pgp-private-keys.asc
gpg --import-ownertrust pgp-ownertrust.asc
```

---

### Deletion / Revocation

#### You messed up? Delete / Revoke your subkey

Something went wrong at the creation, you can delete a subkey? 
Edit your key 

`gpg2 --expert --edit-key E6B0933DD47E3B2F823334F0A567B3B17C27BF6D`

select the `subkey` or `user ID` (depending on what went wrong)

`key F2DF28C8CDF28C`

Then delete the key

`deluid`

**If you dont select the key... well you'll loose all of them**

*Same principle applies for `uid` and `deluid`*

 #### Deletion of a main key 

##### secret key

`gpg--delete-secret-keys BF62354455FWE53355`

(This will also delete all subkeys associated with this secret key)

##### public key

`gpg--delete-keys BFE88984533WETT2233`

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

