# credstash-breakdown

# Goal
1. To explore how credstash works
2. To enable developers to understand how fugue implemented credstash to expand the same concepts to other Key Management Services
# Glossary
1. [AWS KMS (Key Management Service)](http://docs.aws.amazon.com/kms/latest/developerguide/overview.html): AWS Key Management Service (AWS KMS) is a managed service that makes it easy for you to create and control the encryption keys used to encrypt your data **reference to AWS docs**
1. [Symmetric Encryption](https://en.wikipedia.org/wiki/Symmetric-key_algorithm): are algorithms for cryptography that use the same cryptographic keys for both encryption of plaintext and decryption of ciphertext
2. [Cipher](https://en.wikipedia.org/wiki/Ciphertext): is the result of encryption performed on plaintext using an algorithm, called a cipher
3. [Cipher Mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation):  a mode of operation is an algorithm that uses a block cipher to provide an information service such as confidentiality or authenticity
4. [HMAC _Hash-based Message Authentication Code_](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code )
5. [AES _Advanced Encryption Standard_](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard): subset of the Rijandael cipher
6. [SHA _Secure Hash Algorithms_](https://en.wikipedia.org/wiki/Secure_Hash_Algorithms): family of cryptographic hash functions

# References
1. [cryptography](https://cryptography.io/en/latest/): cryptography is a Python library which exposes cryptographic recipes and primitives.
2. http://www.daemonology.net/blog/2009-06-11-cryptographic-right-answers.html
3. https://blog.fugue.co/2015-04-21-aws-kms-secrets.html

## Prerequisites
```
LEGACY_NONCE = b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01'
DEFAULT_DIGEST = 'SHA256'
```

## credstash setup

```
createDdbTable(region=region, table=args.table,
               **session_params)
```
1. Attempts to create a DynamoDB table (default credential-store) in the region provided

#### Things that can go wrong in ```credstash setup```
* The user might not have IAM permissions to Create DynamoDB tables  
* The user might not have IAM permissions to Describe DynamoDB tables

## credstash put

```
def putSecret(name, secret, version="", kms_key="alias/credstash",
              region=None, table="credential-store", context=None,
              digest=DEFAULT_DIGEST, **kwargs):
```
1. Attempts to either auto-version or take a manual version  
2. Gets current boto session  
3. Connect to AWS KMS Service
4. Create an instance of KeyService with the kms connection, kms_key (default is alias/credstash unless overridden by -k flag), and a context (which can be empty)  
5. Call ```seal_aes_ctr_legacy``` with the instance created above, the secret inputed, and digest (DEFAULT_DIGEST by default)  
6. Calls ```generate_key_data``` with 64 bit key _calls [AWS.KMS.generate-data-key](http://docs.aws.amazon.com/cli/latest/reference/kms/generate-data-key.html) as a wrapper_    
        * calls with _alias/credstash_ and gets a CiphertextBlock and Plaintext _blob_  
        * this is essentially a key and an encoded_key  
7. Call to internal ```_seal_aes_ctr``` with the key returned from step 6:bullet2, the secret (as plaintext), LEGACY_NONCE _a binary pad_, and the digest method (again DEFAULT_DIGEST)  
8. Take the key from step 6:bullet2 and split it into a **data_key** and a **hmac_key**  
9. Create CipherInstance with _AES(data_key)_, _CTR(LEGACY_NONCE)_ mode, and the default cryptography backend  
10. Generates an encryptor from the CipherInstance and calls ```encryptor.update()``` with the secret (plaintexted and encoded) and then is locked with finalize()  
11. Calls ```_get_hmac``` with the hmac key,generated from step 7, the ciphertext created from step 9, and the DEFAULT_DIGEST  
12. Generates an HMAC and updates it with the ciphertext created from step 9 and is locked with ```finalize()```  
13. ```_seal_aes_ctr``` returns with the ciphertext and hmac  
14. ```seal_aes_ctr_legacy``` returns with a formatted object that consists of key (b64encoded key from step 6), contents (b64encoded ciphertext from step 12), hmac (hexencoded hmac from step 12), and digest (DEFAULT_DIGEST)  
15. Updates the dictionary from step 13 with name (of the secret's key) and version (that's padded)  
16. Writes to DynamoDB table  

### credstash put *tldr;*
Generate a CiphertextBlock + Plaintext from AWS KMS -> Split CiphertextBlock into data_key and hmac_key -> Encode secret as binary and encrypt with AES (using the CiphertextBlock) using [Counter Method](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Counter_.28CTR.29) -> Generate HMAC -> base64 encode the data_key -> base64 encode the Ciphertext of the encrypted secret -> hex encode the hmac value -> store this inside of DynamoDB with the digest used

#### credstash put *Diagram*

#### Things that can go wrong with ```credstash put```
* [**The user might not have permissions to KMS key**](http://docs.aws.amazon.com/kms/latest/developerguide/control-access.html)
* The user might not have IAM permissions to PutItem into DynamoDB table
* The user doesn't have IAM permission to call KMS.GenerateKeyData

## credstash get
```
def getSecret(name, version="", region=None,
              table="credential-store", context=None,
              **kwargs):
```
1. Determine if version was passed. If not just grab the highest version available
2. Query DynamoDB (_default table credential-store_) for credentials using the version from step 1
3. Extract the name and version from the results from step 2
4. Connect to boto session
5. Connect to kms
6. Pass kms and the extraction from step 3 to ```open_aes_ctr_legacy```
7. Inside of of ```open_aes_ctr_legacy``` we need to decode everything from step 14 in ```credstash put```
    * b64decode key and decrypt with default key unless otherwise put in
        * decrypt is from the KeyService class inside of credstash
        * decrypt calls kms decrypt with the CiphertextBlock as the encoded key and the encryption context (default None)
    * extract the digest method, if None use DEFAULT_DIGEST
    * b64decode the contents
    * hexdcode the hmac value
    * call ```_open_aes_ctr``` with the key decrypted from bullet1, LEGACY_NONCE, the b64dcoded ciphertext from bullet3, the hmac value from bullet 4, and the digest method from bullet 2
7. Inside of ```_open_aes_ctr``` split the key into data_key and hmac_key like step 8 in credstash put
8.Calls ```_get_hmac``` with the hmac key generate from step 7, the ciphertext created from step 6: bullet3, and the DEFAULT_DIGEST  
9. Generates an HMAC and updates it with the ciphertext created from step 9 and is locked with ```finalize()```
10. If HMAC doesn't equal the expected HMAC from the table it's not verified so don't trust
11. Create CipherInstance with _AES(data_key)_, _CTR(LEGACY_NONCE)_ mode, and the default cryptography backend  
12. Decrypt with decryptor from step 11 with the ciphertext from step 6: bullet3 and lock it with ```finalize()```
13. ```open_aes_ctr_legacy``` returns the decrypted value but also decodes it from binary to a string
14. Write out the decrypted / decoded secret to screen

### credstash get *tldr;*
Get the row from DynamoDB based on the secret provided -> base64 decode data_key -> base64 decode contents (the encrypted secret) -> hex decode the hmac -> split the key into data_key and hmac -> generate hmac value -> verify integrity of hmacs -> decrypt using AES with the key and [Counter Method](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Counter_.28CTR.29) -> Decode from binary to string and return
#### credstash get *Diagram*

#### Things that can go wrong with ```credstash get```
* [**The user might not have permissions to KMS key**](http://docs.aws.amazon.com/kms/latest/developerguide/control-access.html)
* The user might not have IAM permissions to call KMS.decrypt
* The user might not have IAM permissions to Query into DynamoDB table
* The user might not have IAM permissions to Scan into DynamoDB table
* The user might not have IAM permissions to GetItem in DynamoDB table
