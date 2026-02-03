# lxp-tools

This repository contains some useful helper scripts for managing CERN LXPlus SSH sessions on both macOS and Linux. In most cases (network instabilities aside), they will bring back the seamless LXPlus experience that we were all used to for so long, without OTP frustrations and mental health breakdowns that come from having to go back to your phone every couple minutes :).

- `lxp`: main high-level tool to manage LXPlus connections, initializing SSH [ControlMaster](https://cern.service-now.com/service-portal?id=kb_article&n=KB0009800) (CM) sessions and Kerberos tokens when necessary. It has shortcuts to login to specific LXPlus machines (e.g. LXTunnel, LXPlus-ARM, or some ATLAS nodes), and VNC port-tunneling. For those of us working in operations at the ATLAS experiment, there are also shortcuts to tunnel Point-1 gateway proxies initialized in LXPlus, in case you need to access the internal P1 network.
- `lcp`: wrapper around `rsync`, automatically resolving the LXPlus hostname and opening Kerberos tickets and SSH CM sessions for the transfer.

**Note**: If a connection is kept open too long (> 24hs), the Kerberos token within the remote node will expire, so attempts to login to the node will take ~10-15 seconds (until the attempt to lock `.Xauthority` times out), and calls to `lcp` using that node will fail. You also won't be able to access any file that requires an authentication (AFS or EOS). You can easily solve this by calling `kinit` within the node (after waiting for the lock timeout), and manually providing your password. You can avoid this issue entirely by keeping a [(CERN-customized) tmux instance](https://hsf-training.github.io/analysis-essentials/shell-extras/persistent-screen.html) running.

A few low-level tools used internally by the main scripts are also included:
- `kerb`: initializes and auto-renews Kerberos tokens.
- `sshcm-otp`: initializes and manages SSH CM sessions to LXPlus, including the 2-factor authentication OTP retrieval from keys stored in a secure GPG keychain.
- `lxhost`: builds the LXPlus hostname from mnemonics and other shortcuts.


Contributions with shortcuts/aliases specific for ATLAS or other CERN experiments are welcomed!

For questions and/or suggestions, please contact me on Mattermost or to `jean.yves.beaucamp@cern.ch`.


## Requirements

The included tools depend on the following auxiliary packages
- [krb5](https://web.mit.edu/kerberos) (no need to install a custom version in macOS, the built-in one works),
- [pass](https://www.passwordstore.org),
- [pass-otp](https://github.com/~/pass-otp),
- [expect](https://www.nist.gov/services-resources/software/expect),
- [rsync](https://rsync.samba.org) (on macOS, you'll also need an updated `rsync` version, which you can install from `brew`).

Optional (and recommended):
- [kstart](https://www.eyrie.org/~eagle/software/kstart): modified version of `kinit` (`k5start`) which runs as a deamon, to maintain the ticket for a maximum of 7 days (with the CERN configuration).

Some will be pre-installed, and the remaining ones should be available through `brew` on macOS, or using your Linux distribution package manager.


## Setup

### Installation

The bundled scripts have no installer, so we suggest to clone the repository in `~/.local/bin`, and add its directory to your local `PATH`.

Since for most people your CERN username will be different from your local username, the `$CERN_USERNAME` environment variable is used if available.

Therefore, you should add the following lines (replacing your username) to your local `.zshrc`, `.bashrc`, or equivalent:
```
export CERN_USERNAME="username"
export PATH="$HOME/.local/bin/lxp-tools/scripts:$PATH"
```

To enable tab-completion of remote paths for `lcp` in zsh, include the following additional lines in `.zshrc`
```
fpath=("$HOME/.local/bin/lxp-tools/autocomplete/zsh" $fpath)
autoload -U compinit && compinit # Only needed if you are NOT using Oh-My-Zsh
```
If you are using Oh-My-Zsh, the modification of the `fpath` array has to be inserted at any point **before** the initialization of Oh-My-Zsh (`source $ZSH/oh-my-zsh.sh`). In both cases, you will need to delete all `.zcompdump*` files in your home directory and open a new zsh session (or run `source ~/.zshrc`) to enable this functionality the first time, or when updating this package.


### Kerberos

In order to use Kerberos to authenticate with your CERN account, you'll first need to setup the correct configuration in your local client (`krb5.conf`). Follow the official CERN instructions [here](https://linux.web.cern.ch/docs/kerberos-access/#client-configuration-kerberos).

To avoid having to introduce your password every time, you'll need to store your Kerberos credentials in a secure local storage. This is done through a _keytab_ file, which can be [automatically generated](https://cern.service-now.com/service-portal?id=kb_article&n=KB0003405) with the correct encryption settings on any LXPlus node running EL9 or higher. Simply, run
```
[lxplus] $ cern-get-keytab --keytab .keytab --user --login username
```
replacing your CERN username after `--login`, and copy the file to your local machine with `scp`
```
$ scp username@lxplus.cern.ch:.keytab ~/.keytab
```
The keytab must only be readable by you. The correct permissions should be set by `cern-get-keytab` and carried over through `scp`, but always double check this. If you update your CERN credentials, you'll need to re-generate and copy the keytab.

**Note**: The built-in version of `kinit` in macOS stores passwords in the macOS encrypted keychain when successfully initializing a Kerberos token with `--keychain`, so you could just use that. However, the tools in this repository will use `k5start` if available, which does not support this feature, so setting up a keytab is recommended.


### SSH client

You'll need to update the configuration of your local SSH client, to enable the delegation of Kerberos tokens and the use of ControlMaster sessions. The minimum required settings look like this
```
HOST lxplus*
    GSSAPIAuthentication yes        # Kerberos auth
    GSSAPIDelegateCredentials yes   # Kerberos ticket delegation
    ControlPath ~/.ssh/%r@%h:%p     # CM session socket location (for macOS)
    ControlMaster auto              # Always use CM sockets for new SSH connections
    ControlPersist yes              # Persist the socket after the first session is destroyed
    
HOST lxtunnel*
    # Same here
```
In Linux installations, change the location for CM session sockets to `/run/user/%i/%r@%h:%p` (required for the included scripts).

Usually, a few more options will be needed, so a day-to-day optimized configuration will be
```
HOST lxplus*
    User username
    ForwardX11 yes                  # Enable X11 auto-forwarding
    ForwardX11Trusted no            # To be on the safe side just in case
    HashKnownHosts yes              # Store a list of host hashes instead of IPs locally, for safety
    GSSAPIAuthentication yes        # Kerberos auth
    GSSAPIDelegateCredentials yes   # Kerberos ticket delegation
    PubkeyAuthentication no         # Disable public key authentication (useless in LXPlus in most cases)
    ForwardAgent yes                # Forward public keys when logging in to other hosts from within LXPlus
    CheckHostIP no                  # Ignore IP addresses, since they move a lot within the CERN network
    ServerAliveInterval 100         # Keep the server alive even during short downtimes
    ControlPath ~/.ssh/%r@%h:%p     # CM session socket location (for macOS)
    ControlMaster auto              # Always use CM sockets for new SSH connections
    ControlPersist yes              # Persist the socket after the first session is destroyed
    
HOST lxtunnel lxtunnel.cern.ch
    Hostname lxtunnel.cern.ch
    User username
    HashKnownHosts yes
    GSSAPIAuthentication yes
    GSSAPIDelegateCredentials yes
    PubkeyAuthentication no
    ForwardAgent yes
    CheckHostIP no
    ServerAliveInterval 100
    ControlPath ~/.ssh/%r@%h:%p
    ControlMaster auto
    ControlPersist yes
    Protocol 2                      # Use the SSH protocol version 2
    DynamicForward 8090             # Port forward to ATLAS CI/build Jenkins nodes
```


### 2FA configuration

To avoid having to reach for your phone every time you have to initialize a new SSH CM session to LXPlus, you can store the 2FA `otpauth` master key encrypted in a GPG key on your local machine, and generate new OTPs transparently. As long as you encrypt and follow safe practices on your GPG key, this is just as safe as using TouchID in the cern SSO web-authentication. You also need to have the correct global time in your local computer (same as in your phone if using OTP apps)

The first step is to retrieve the `otpauth` master key for your CERN account. You can export them from the Authenticator app you've been already using. In case the app only allows the export in the form of a QR code (e.g. Google Authenticator), you can either import the QR into another OTP applicaiton that can export `otpauth` URIs, or you can use [this tool](https://github.com/scito/extract_otp_secrets).

Before importing the keys into `pass-otp`, if you're not already using `pass`, you'll need to initialize the GPG secret key that will be used as the secure storage. For this, run
```
$ gpg --full-generate-key
```
and use all the default options. On Macs with TouchID support, you may use [pinentry-touchid](https://github.com/jorgelbg/pinentry-touchid). If you're using a YubiKey, follow the [official instructions](https://support.yubico.com/s/article/Using-Your-YubiKey-with-OpenPGP) to generate a key locked by your 2FA hardware key. 

Now, get the `uid` for your created key from
```
$ gpg --list-keys
```
and use that `uid` to initialize the pass keychain storage, e.g.
```
$ pass init "Your Name Here <address@email.com>"
```
Copy the string from the `gpg --list-keys` output, including all spaces between within your full name, and the `<` and `>` surrounding the email address.

Finally, insert the `otpauth` secret key for your CERN account
```
$ pass otp insert cern-username otpauth://totp/...
```
The name of the key must be of the form `cern-${username}` to work with the scripts in the repository.

You get updated OTPs at any time with
```
$ pass otp cern-username
```
This is very useful to login to the CERN SSO in machines where you don't have a WebAuth token.
