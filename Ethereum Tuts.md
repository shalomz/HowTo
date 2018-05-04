# Tutorial: Controlling Ethereum with Python

Ethereum makes it easy to run simple computations on blockchains while
[keeping the value of decentralized networks](http://ecomunsing.com/what-are-blockchains-good-for)- but real applications will need access to off-
blockchain resources, whether for heavy computations or pulling data from the
internet. This tutorial walks through how to deploy a smart contract on a
private test blockchain and use Python to read data from the blockchain or
send data to the blockchain. We'll review the relevant components of the
Ethereum network, walk through how to interact with the system using Python,
and deploy example smart contracts.

Note that this was written in July 2016 using the Ethereum Homestead release,
Python 2.7.12, and geth 1.6.6. More resources are found on the official
[Ethereum website](https://ethereum.org/) and the [Ethereum Homestead documentation](http://www.ethdocs.org/en/latest/). Because these projects are
open-source, there is a lot of outdated information (probably including this
guide, by the time you read it). This notebook is also available on github
here.

## Background: Ethereum is not a computer[¶](http://ecomunsing.com/tutorial-controlling-ethereum-with-python#Background:-Ethereum-is-not-a-computer)

For background, it's helpful to keep in mind that "Ethereum" is a
communication protocol- not a specific network, computer, or application. Just
as the main Internet exists alongside many private Intranets and local
networks, there is a main community Ethereum network (which supports the Ether
cryptocurrency) alongside a community test network (called "Ropsten") and many
private networks. For this tutorial we'll be working on implementing a test
server that just serves data on your local computer- just as you might run a
`localhost://` server for developing a web page, but later deploy it to a
wider network.

For my research, I don't want to deploy on the full Ethereum network, where my
code is subject to attack and operations cost real Ether- instead ther are a
variety of different options for prototyping:

  * Private node in development mode: Just run an ethereum node on your own machine, ignoring all connections- the equivalent of running on localhost.
  * Private network: Networking a set of computers into a private network that isn't discoverable by others.
  * Public test network: Open for anyone to access, but configured to quickly generate Ether for testing software.
  * Full Ethereum network: This is the real deal, where Ether costs money and hackers may roam.

So Ethereum is a communication protocol- but how do we interact with it? There
are different open-source software packages which connect with Ethereum
networks for mining or development purposes- these are equivalent to competing
internet browsers. Common implementations are `eth` ([the C++ implementation](https://github.com/ethereum/cpp-ethereum), not to be confused
with the cryptocurrency), and `geth` ([the GoLang implementation](https://github.com/ethereum/go-ethereum), which we'll be using here).

Ultimately, we care about the scripts which are run on the Ethereum Virtual
Machine (the EVM)- the decentralized, consensus-driven computer which
distinguishes Ethereum from earlier blockchains. This Virtual Machine runs its
own language of bytecode, which we'll generate with a compiler. We'll be
writing scripts in [Solidity](http://solidity.readthedocs.io/), a Javascript-
derived language, and compiling them into bytecode using the `sol-c` compiler.
We'll then use our Ethereum client (i.e `geth`) to push this compiled bytecode
to the Ethereum network, and to run the bytecode which others have written.

From now on, we'll assume that we're working with the `geth` implementation
and will work on creating a private test network on our own computer in order
to develop and deploy smart contracts which we'll be interacting with using
Python.

For development, there are 3 pieces of software that you'll interact with: the
Ethereum node (the server), the console (where you can type code to interact
directly with the server), and the Ethereum Wallet (a GUI which provides
graphical interaction with the server. These are configured with two important
directories: the datadir which stores the important configuration, and the
location of your .ipc (inter-process communication) file which lets the pieces
talk with each other.

### Important software pieces:[¶](http://ecomunsing.com/tutorial-controlling-ethereum-with-python#Important-software-pieces:)

  * Ethereum Node (e.g. Geth server): This runs your actual Ethereum node and maintains a connection with the rest of the Ethereum network. This can be created by running `geth` (starts server only), by running `geth console` (starts server and then console), or by opening the Ethereum Wallet. However you start it, the ethereum node is discoverable to related software through 
  * Console: Allows Javascript command-line interface to the server. Can be used for changing settings or deploying contracts. Run by `geth console` (will attempt to spin up a node) or `geth attach` (will look for geth.ipc file)
  * Ethereum Wallet (aka Mist): GUI for interfacing with wallets and deploying contracts. Will look for /Users/emunsing/Library/Ethereum/geth.ipc and if it does not find geth.ipc the Wallet will start up a server for the node.

**Configuration**  
The Ethereum environment is dictated by two data sources: the data directory
(called 'datadir' here) and the `geth.ipc` file. The datadir stores the
blockchain, account keys, and other personal settings. The `geth.ipc` file is
created when a node starts up, and allows other services (geth console,
ethereum wallet) to attach to that server. In addition, there are a number of
command line options that you can use to direct how and where your Ethereum
node should look for peers.

  * By default, the Ethereum Wallet or `geth attach` looks for the geth.ipc file in `~/Library/Ethereum/geth.ipc` (i.e. `/Users/username/Library/Ethereum/geth.ipc` on Mac OSX).
  * Specify the datadir location with the command line options `--datadir ~/data` and specify the ipc location with `--ipcpath ~/Ethereum/geth.ipc`
  * Generally avoid using `geth.ipc` locations other than the default- there is no easy way to configure the Wallet to connect to these  
If you're using a computer for developing new ethereum apps, you might want to
have a number of different blockchains for different public and private
implementations. I find it helpful to keep these separate by running one node
per computer, using the default .ipc location, and using separate datadir
locations for each project.

### Back up the /keystore![¶](http://ecomunsing.com/tutorial-controlling-ethereum-with-python#Back-up-the-/keystore!)

Each account's private keys are encrypted and stored in the /keystore folder
of the datadir. These should be backed up off of the hard drive in case of
removal- whether accidental or malicious.

  * To restore an account after /keystore was removed, simply paste the backed-up files back to `/keystore` under the data directory
  * If trying to just clean up the blockchain, can do so with `geth upgradedb` or `geth removedb` - these will not affect the keystore

## Installing Geth[¶](http://ecomunsing.com/tutorial-controlling-ethereum-with-python#Installing-Geth)

Geth, or go-ethereum, is the main software platform for running an ethereum
node on your computer. You can use this to connect to the main ethereum
network, the public test network, or to create your own private test network.

We'll install Geth using Homebrew, following [the Geth installation
instructions for Mac](https://github.com/ethereum/go-ethereum/wiki/Installation-Instructions-for-Mac). This tutorial was written using

  * Geth v1.6.6
  * Python v2.7.12
  * ethjsonrpc v0.3.0
  * py-solc v.1.2.0

First, make sure that homebrew is installed and up-to-date by running `brew
update` …if it isn't installed, [follow theseinstructions](https://www.howtogeek.com/211541/homebrew-for-os-x-easily-installs-desktop-apps-and-terminal-utilities/).

    
    
    brew update brew tap ethereum/ethereum brew install ethereum 

Check that Geth is now installed by running `geth account new` which will
prompt you to create a new account.

To upgrade Geth, run `brew update` &amp;&amp; `brew upgrade ethereum`. You can
check your version with `geth version`

## Installing Sol-c (compiler)[¶](http://ecomunsing.com/tutorial-controlling-ethereum-with-python#Installing-Sol-c-\(compiler\))

  * Install the solidity compiler: npm install -g solc or npm install solc -global then check that the installation succeeded: solcjs -help
  * Install solC using homebrew 
    * Get the directory solc installed to: `$ which solc`
    * Copy this directory, and use it in the geth console like `> admin.setSolc("/usr/local/bin/solc")`
    * Confirm that the compiler has been successfully installed by running `> eth.getCompilers()`

## Installing Ethereum Wallet (optional)[¶](http://ecomunsing.com/tutorial-controlling-ethereum-with-python#Installing-Ethereum-Wallet-\(optional\))

There are two common ways to interface with an Ethereum network: through the
command line, and with the Wallet GUI app (also called 'Mist'). We'll be using
the command line for coding, but you might find it helpful to use the Wallet
app to explore what's going on in your Ethereum network and make sure that
everything is running as expected. The Wallet is designed to provide easy
usability out-of-the-box, and [installers can be found
here](https://github.com/ethereum/mist/releases). Some notes:

  * The Wallet app will look for a `geth.ipc` file in the default directory, and if it doesn't find one will start the loooong process of syncing with the main Ethereum network. If you don't want this on the main network, be sure to run an Ethereum node from the command line ahead of time.
  * By default, an etherbase account is created. This is an externally owned account, not a wallet contract- i.e. it is not visible on the blockchain.
  * To get enough ether to create a transaction, start CPU mining (Menu&gt; ) or use a _faucet_ [like this one](http://faucet.ropsten.be:3001/) to request test Ether be sent to your account (this is "play money", as you're on the test network).

## Install required Python packages:[¶](http://ecomunsing.com/tutorial-controlling-ethereum-with-python#Install-required-Python-packages:)

I'll be using a virtual environment running Python 2.7.12, and install the
required packages for interfacing with Ethereum:

  * `pip install py-solc`
  * ~~`pip install ethjsonrpc`~~ **NOTE**: Due to changes in how Geth handles RPC calls, the pip version of ethjsonrpc (v0.3.0) does not currently work, and . I've made a [working fork and here](https://github.com/emunsing/ethjsonrpc)- for this demo, it is sufficient but necessary to be able to run the example in README.md. You can clone my fork and set it up by running `$ python setup.py install`

From here on, we'll use the dollar sign to indicate Unix/bash command line
entries like `$ geth attach` and use the angle bracket to indicate commands
entered into the geth console like `> miner.start(1)`

Setting up a private test network:

  1. Run a server node with `$ geth --dev --rpc --ipcpath ~/Library/Ethereum/geth.ipc --datadir ~/Library/Ethereum/pyEthTutorial`
  2. In a second terminal window, attach a geth console with `$ geth attach`
  3. If you haven't created an account yet, you'll need to create a new account with `> personal.newAccount()` and follow the prompt to create a password. If everything is running right, you should see your `geth` server output announce the creation of a new account, and should also see a new file appear in your /keystore directory. 
  4. You can get a list of all the accounts associated with your keystore by running `> eth.accounts`. We'll assume that we want to work with the first of these, i.e. `> eth.accounts[0]`
  5. In the geth console, unlock the coinbase account with `> personal.unlockAccount( eth.accounts[0], 'passwd', 300)` where 300 is the number of seconds for which it should be unlocked (set to 0 for indefinite). The console should output **`true`**
  6. Start mining with `> miner.start(1)` which should trigger a stream of text on your first terminal window. You can now see your balance in Ether by checking `web3.fromWei( eth.getBalance(eth.coinbase), "ether")`
