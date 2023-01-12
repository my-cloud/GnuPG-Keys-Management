# GnuPG Keys Managment





## Goal

Create a master key with Certification capability and then 3 subkeys with these capabilities:  Sign, Encrypt ant Authenticate.

Then for increasing the secutity, the master private key must be stored on a encrypted storage device and deleted from the computer.



## Create gpg key pair



create a template:

```bash
cat <EOF> primary_gpg_key_constructor.tpl
%echo Generating a RSA key 8192
Key-Type: RSA
Key-Length: 8192
Key-Usage: cert,sign,auth
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

(`eanble-lage-rsa` is to have a key of size over 4096 bit)

#### Check your private keys

`gpg --list-secret-keys`

#### Check your public keys

`gpg --list-keys`



## Create Subkeys

Edit your key 

`gpg --expert --edit-key 123456789123456789123456789`

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

Select 4 and validate. (To create a sign only RSA 4096 bits)

Add another key with `addkey`, select 6 and validate

Add the last key with `addkey`, select 8 then A then Q and validate

Then exit gpg nicely by entering `q` then `y`

#### Check all the Subkeys

` gpg --fingerprint --fingerprint 123456789123456789123456789`



## Deletion

Something went wrong at the creation, you can delete a subkey? 
Edit your key 

`gpg2 --expert --edit-key 123456789123456789123456789`

select the `subkey` or `user ID` (depending on what went wrong)

#### exemple for a subkey
**Select the subkey!**

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

## Revocation

One of your subkey has been compromised, let everyone know!

**Select the subkey!**

`key F2DF28C8CDF28C`

Then delete the key

`revkey`



## Protect your master key

In order to protect your master key, you need to remove it from the local computer. Before to do so, you need to securely store the master key on an encrypted external usb device. 

*In the usb drive, create a folder `secret`. *
*Optionally, set a local versioning with `git init` in this folder.*

(The reason to create a local git is to track change on the `secret` folder )

Below are the commands to save the master key:

```bash
# Change the values of these variables if needed
export EXPORT_DIR="<KeyLocation>/secrets/gpg"
export EXPORT_FILENAME="MySelf_Master_MyComment_private_rsa8192"
export EXPORT_KEY_ID=1231231232123123213123
mkdir -p ${EXPORT_DIR}
gpg --armor --output $EXPORT_DIR/$EXPORT_FILENAME.key.sec --export-secret-keys $EXPORT_KEY_ID
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

select 1 (Key has been compromised)

select 2 (key is superseded)

select 3 (key is no longer used)



## Sign

You can sign the new master key with your previous key that you're transitioning from.

`gpg --sign-key <longid>`



## Master key deletion

To increase the security, it is advised to remove the main key, and to only use the subkeys.

list the keygrip

```
gpg2 --list-secret-keys --with-keygrip
/home/alice/.gnupg/pubring.kbx
------------------------------
sec   rsa4096/CB2F38F25B491A54 2014-12-31 [C] [expires: 2017-12-30]
      Keygrip = D4DF0C35D3E22FA6AC37DA2E54FB03F73616A3CB
uid               [ultimate] Alice <alice@example.org>
ssb   rsa2048/04BB7F8FDEC5E5D9 2014-12-31 [S] [expires: 2015-12-31]
      Keygrip = 21B2EDF018D7CAF0B45644FDB753DD42307C4425
ssb   rsa2048/BBB6B86627C2D43A 2014-12-31 [E] [expires: 2015-12-31]
      Keygrip = 2E149DA9C5E46E0DECC6A17EFD8B5FB1DF1E1BAB
ssb   rsa2048/7D2233B8833E70AF 2014-12-31 [A] [expires: 2015-12-31]
      Keygrip = 22149DA9C44FRT785FH345FEFD8B5FB1DF112B55
```



Then you can remove the Master private key (specifying the keygrip id to delete)

```
gpg-connect-agent "DELETE_KEY D4DF0C35D3E22FA6AC37DA2E54FB03F73616A3CB" /bye
```



Ensure that the private master key has been removed

```
gpg2 --list-secret-keys
/home/alice/.gnupg/pubring.kbx
------------------------------
sec#  rsa4096/CB2F38F25B491A54 2014-12-31 [C] [expires: 2017-12-30]
uid               [ultimate] Alice <alice@example.org>
ssb   rsa2048/04BB7F8FDEC5E5D9 2014-12-31 [S] [expires: 2015-12-31]
ssb   rsa2048/BBB6B86627C2D43A 2014-12-31 [E] [expires: 2015-12-31]
ssb   rsa2048/7D2233B8833E70AF 2014-12-31 [A] [expires: 2015-12-31]
```



you should see the # symbol after the sec keyword, confirming that the private master key is not usable.





## Use the offline master key

Yubikey : https://www.youtube.com/watch?v=xGsixSh6sC4



otherwise

```
mkdir -p ~/gpgtmp
chmod 0700 ~/gpgtmp
gpg2 --homedir ~/gpgtmp --import /run/media/alice/mystick/alice-private-keys.asc
gpg2 --homedir ~/gpgtmp --keyring ~/.gnupg/pubring.kbx --edit-key bob@example.com
gpg-connect-agent --homedir ~/gpgtmp KILLAGENT /bye
rm -rf ~/gpgtmp

```



## Import someone's public key:

```
gpg --import pgp-public-keys.asc
```

The `pgp-public-keys.asc` is the key name of that person.

## Reimport

in case you ever need to reimport :

```
gpg --import pgp-public-keys.asc
gpg --import pgp-private-keys.asc
gpg --import-ownertrust pgp-ownertrust.asc
```



## Encrypt file

```
gpg --encrypt --sign --armor -r receipient@gmail.com message.txt
```

The `-encrypt` option tells gpg to encrypt the file, and the `--sign` option tells it to sign the file with your details. The `--armor` option tells gpg to create an ASCII file. The `-r` (recipient) option must be followed by the email address of the person you’re sending the file to.

The `message.txt` is the name of the file.

The file is created with the same name as the original, but with “.asc” appended to the file name (ex. `message.asc`)

## Decrypt file

`gpg --decrypt coded.asc`

The `coded.asc` is the file received. 

To redirect the output into another file :

`gpg --decrypt coded.asc > plain.txt`





## References

https://incenp.org/notes/2015/using-an-offline-gnupg-master-key.html

https://gist.github.com/ageis/5b095b50b9ae6b0aa9bf

https://riseup.net/en/security/message-security/openpgp/gpg-best-practices

https://www.devdungeon.com/content/gpg-tutorial

https://8gwifi.org/docs/gpg.jsp

