# Yubikey as an SSH key

All other guides I've seen (https://github.com/drduh/YubiKey-Guide being the most prolific) tell you to use the Yubikey's smartcard (PKCS#11) features with GnuPG via gpg-agent.

STOP THE MADNESS!

OpenSSH has supported OpenSC since version 5.4. This means that all you need to do is install the OpenSC library and tell SSH to use that library as your identity.

## How?

I did this all on macOS 10.14. Linux distributions should work in a similar way. This is based on [Yubico's instructions](https://developers.yubico.com/PIV/Guides/SSH_with_PIV_and_PKCS11.html) but uses the newer `ykman` utility instead of the older `yubico-piv-tool`. The older tool doesn't seem to support generating PIV certificates and gives [misleading errors](https://github.com/Yubico/yubico-piv-tool/issues/153).

1. Install OpenSC

    brew install opensc

2. Install Yubikey manager (CLI only)

    brew install ykman

3. Ensure CCID mode is enabled on the Yubikey

    ykman mode

If CCID is not in the list, enable it by adding CCID to the list, e.g.

    ykman mode OTP+FIDO+CCID

(This assumes you had OTP+FIDO previously, and still want them enabled.)

4. Generate a PIV key and output the public key

    ykman piv generate-key 9a pubkey.pem

This is an RSA 2048-bit key by default. Depending which Yubikey you have, you can change it using `-a` / `--algorithm`.

(9a is the PIV authentication slot.)

5. Generate a self-signed X.509 certificate

    ykman piv generate-certificate -s "SSH key" 9a pubkey.pem

6. Export your SSH public key from the Yubikey

    ssh-keygen -D /usr/local/lib/opensc-pkcs11.so

And that's all the hard stuff done. 

Now just add the public key to your `authorized_keys` file on a remote host and try to use it:

    ssh -I /usr/local/lib/opensc-pkcs11.so -i /usr/local/lib/opensc-pkcs11.so -o IdentitiesOnly=yes server.example.com

You should be prompted for your Yubikey's PIV PIN.