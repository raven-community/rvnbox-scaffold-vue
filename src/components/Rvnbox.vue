<template>
  <div class="App">
    <header class="App-header">
      <img src="../assets/logo.png">
      <h1 class="App-title">Hello RVNBOX</h1>
    </header>
    <div class='App-content'>
      <h2>BIP44 $RVN Wallet</h2>
      <h3>256 bit {{lang}} BIP39 Mnemonic:</h3> <p>{{mnemonic}}</p>
      <h3>BIP44 Account</h3>
      <p>
        <code>
        "m/44'/175'/0'"
        </code>
      </p>
      <h3>BIP44 external change addresses</h3>
      <ul>
        <li v-for="(address, index) in addresses" :key="index">m/44&rsquo;/175&rsquo;/0&rsquo;/0/{{index}}:{{address}}</li>
      </ul>
      <h3>Transaction raw hex</h3>
      <p>{{hex}}</p>
    </div>
  </div>
</template>

<script>
let RVNBOXSDK = require('rvnbox-sdk/lib/rvnbox-sdk').default
let RVNBOX = new RVNBOXSDK()

let network = "testnet";

let langs = [
  'english',
  'chinese_simplified',
  'chinese_traditional',
  'korean',
  'japanese',
  'french',
  'italian',
  'spanish'
]

let lang = langs[Math.floor(Math.random() * langs.length)]

// create 256 bit BIP39 mnemonic
let mnemonic = RVNBOX.Mnemonic.generate(256, RVNBOX.Mnemonic.wordLists()[lang])

// root seed buffer
let rootSeed = RVNBOX.Mnemonic.toSeed(mnemonic)

// master HDNode
let masterHDNode = RVNBOX.HDNode.fromSeed(rootSeed, network)

// HDNode of BIP44 account
let account = RVNBOX.HDNode.derivePath(masterHDNode, "m/44'/175'/0'")

// derive the first external change address HDNode which is going to spend utxo
let change = RVNBOX.HDNode.derivePath(account, '0/0')

// get the Legacy address
let LegacyAddress = RVNBOX.HDNode.toLegacyAddress(change)

let hex
let txid

RVNBOX.Address.utxo(LegacyAddress).then(
  result => {
    if (!result[0]) {
      return
    }

    // instance of transaction builder
    let transactionBuilder = new RVNBOX.TransactionBuilder(network)
    // original amount of satoshis in vin
    let originalAmount = result[0].satoshis

    // index of vout
    let vout = result[0].vout

    // txid of vout
    txid = result[0].txid

    // add input with txid and index of vout
    transactionBuilder.addInput(txid, vout)

    // get byte count to calculate fee. paying 1 sat/byte
    let byteCount = RVNBOX.RavenCoin.getByteCount({ P2PKH: 1 }, { P2PKH: 1 })
    // 192
    // amount to send to receiver. It's the original amount - 1 sat/byte for tx size
    let sendAmount = originalAmount - byteCount

    // add output w/ address and amount to send
    transactionBuilder.addOutput(LegacyAddress, sendAmount)

    // keypair
    let keyPair = RVNBOX.HDNode.toKeyPair(change)

    // sign w/ HDNode
    let redeemScript
    transactionBuilder.sign(
      0,
      keyPair,
      redeemScript,
      transactionBuilder.hashTypes.SIGHASH_ALL,
      originalAmount
    )

    // build tx
    let tx = transactionBuilder.build()
    // output rawhex
    hex = tx.toHex()

    // sendRawTransaction to running RVN node
    RVNBOX.RawTransactions.sendRawTransaction(hex).then(
      result => {
        txid = result
        console.log(result)
      },
      err => {
        console.log(err)
      }
    )
  },
  err => {
    console.log(err)
  }
)

let addresses = []
for (let i = 0; i < 10; i++) {
  let childNode = masterHDNode.derivePath(`m/44'/175'/0'/0/${i}`)
  addresses.push(RVNBOX.HDNode.toLegacyAddress(childNode))
}

export default {
  name: 'RvnBox',
  data() {
    return {
      mnemonic: RVNBOX.Mnemonic.generate(256),
      lang: lang,
      addresses: addresses,
      hex: hex
    }
  }
}
</script>

<style scoped>
.App {
  font-family: sans-serif;
  max-width: 100%;
  word-wrap: break-word;
}
</style>
