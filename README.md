# Cardano NFT Minter

This is a create react app and can be launched by running `npm install` and `npm start` from the root directory

### Purpose
To mint NFTs on the Cardano Blockchain using the `cardano-serialization-lib` which means it can be built in the browser, without the need to interact with smart contracts

### Workflow minting in Browser - work in progress

- Select image to upload from user's computer
- The app loads the image onto IPFS
- The app generates ...

# Steps to mint an NFT using cardano-cli

Remastered from [Cardano Portal](https://developers.cardano.org/docs/native-tokens/minting-nfts/)

- Register on [Pinata IPFS gateway](https://www.pinata.cloud/) , or similar and upload your image
- You will receive a code where your image is now hosted on IPFS
- Example code: `https://ipfs.io/ipfs/QmPanjt7mQWKmySWddGSvx8iyRDWNSGVn4W3KqHGcE4KiU`

```shell
realtokenname="GUINCHO"
tokenname="4755494e43484f"
#tokenname=$(echo -n $realtokenname | xxd -b -ps -c 80 | tr -d '\n')
tokenamount="1"
fee="0"
output="0"
ipfs_hash="QmPanjt7mQWKmySWddGSvx8iyRDWNSGVn4W3KqHGcE4KiU"
```

### Generate keys
```shell
cardano-cli address key-gen --verification-key-file payment.vkey --signing-key-file payment.skey
cardano-cli address build --payment-verification-key-file payment.vkey --out-file payment.addr --testnet-magic 1097911063
address=$(cat payment.addr)
```

### Fund the address
Send the funds to the address.addr.

You can get testADA from the [FAUCET](https://testnets.cardano.org/en/testnets/cardano/tools/faucet/)

Check that funds have been received by running the following command
```shell
cardano-cli query utxo --address $address --testnet-magic 1097911063
```

### Export protocol parameters
```shell
cardano-cli query protocol-parameters --testnet-magic 1097911063 --out-file protocol.json
```

### Create policyId
```shell
cardano-cli address key-gen \
    --verification-key-file policy.vkey \
    --signing-key-file policy.skey
```

Find the current `slot` and add 43200 to it (so that the NFT can only be minted between now and the next 43 200 seconds, or about 12 hours)
```shell
cardano-cli query tip --testnet-magic 1097911063
slotnumber=<slot number>
```
Create the policy.script file
```shell
echo "{" >> policy.script
echo "  \"type\": \"all\"," >> policy.script 
echo "  \"scripts\":" >> policy.script 
echo "  [" >> policy.script 
echo "   {" >> policy.script 
echo "     \"type\": \"before\"," >> policy.script 
echo "     \"slot\": $(expr $slotnumber + 43200)" >> policy.script
echo "   }," >> policy.script 
echo "   {" >> policy.script
echo "     \"type\": \"sig\"," >> policy.script 
echo "     \"keyHash\": \"$(cardano-cli address key-hash --payment-verification-key-file policy.vkey)\"" >> policy.script 
echo "   }" >> policy.script
echo "  ]" >> policy.script 
echo "}" >> policy.script
```

```shell
script="policy.script"
```

```shell
cardano-cli transaction policyid --script-file policy.script > policyID
```

### Create Metadata

```shell
echo "{" >> metadata.json
echo "  \"721\": {" >> metadata.json 
echo "    \"$(cat policyID)\": {" >> metadata.json 
echo "      \"$(echo $realtokenname)\": {" >> metadata.json
echo "        \"description\": \"Guincho in Cascais, Portugal, overlooking the Sintra mountain\"," >> metadata.json
echo "        \"name\": \"Guincho NFT\"," >> metadata.json
echo "        \"id\": \"1\"," >> metadata.json
echo "        \"image\": \"ipfs://$(echo $ipfs_hash)\"" >> metadata.json
echo "      }" >> metadata.json
echo "    }" >> metadata.json 
echo "  }" >> metadata.json 
echo "}" >> metadata.json
```

### Build the transaction
Check that you have some ADA balance at your address
```shell
cardano-cli query utxo --address $address --testnet-magic 1097911063
```
Set the TxHash, TxIx and Amount to variables:
```shell
txhash="1d41be2e9e3fbdb658871a4c2673883929b3e439f2f30bca4d5b3646854ede66"
txix="0"
funds="10000000"
policyid=$(cat policyID)
output=1400000
```
check that all the variables have been set
```shell
echo $fee
echo $address
echo $output
echo $tokenamount
echo $policyid
echo $tokenname
echo $slotnumber
echo $script
```
build the transaction
```shell
cardano-cli transaction build \
--testnet-magic 1097911063 \
--alonzo-era \
--tx-in $txhash#$txix \
--tx-out $address+$output+"$tokenamount $policyid.$tokenname" \
--change-address $address \
--mint="$tokenamount $policyid.$tokenname" \
--minting-script-file $script \
--metadata-json-file metadata.json  \
--invalid-hereafter $(expr $slotnumber + 43200) \
--witness-override 2 \
--out-file matx.raw
```
if the transaction gives an error and says something like `Minimum required UTxO: Lovelace 1448244`, then change your `output` amount to that larger number
After successfully running it should show you the estimated transaction fee `Estimated transaction fee: Lovelace 187941`

### Sign the transaction
```shell
cardano-cli transaction sign  \
--signing-key-file payment.skey  \
--signing-key-file policy.skey  \
--testnet-magic 1097911063 \
--tx-body-file matx.raw  \
--out-file matx.signed
```

### Submit the transaction
```shell
cardano-cli transaction submit --tx-file matx.signed --testnet-magic 1097911063
```
If you get `Transaction successfully submitted.` then it means that everything worked as planned

### Check that you NFT has been minted
If you minted on the mainnet then search for your address at `pool.pm`
If you minted on the testnet then search for the policyId at `https://testnet.adatools.io/nft`