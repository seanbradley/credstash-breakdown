# credstash-breakdown

http://www.daemonology.net/blog/2009-06-11-cryptographic-right-answers.html

https://blog.fugue.co/2015-04-21-aws-kms-secrets.html

# Glossary
1) Symmetric Encryption
2) Cipher
3) [HMAC _Hash-based Message Authentication Code_](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code )

# Basic Troubleshooting

# Principles

# Background

## Prereqs
```
LEGACY_NONCE = b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01'
DEFAULT_DIGEST = 'SHA256'
```


# HMAC (Hash-based Message Authentication Code)
Hash-based message authentication codes (or HMACs) are a tool for calculating message authentication codes using a cryptographic hash function coupled with a secret key. You can use an HMAC to verify both the integrity and authenticity of a message. (insert reference to https://cryptography.io/en/latest/hazmat/primitives/mac/hmac/)


## credstash setup

```
createDdbTable(region=region, table=args.table,
               **session_params)
```

## credstash put

```
def putSecret(name, secret, version="", kms_key="alias/credstash",
              region=None, table="credential-store", context=None,
              digest=DEFAULT_DIGEST, **kwargs):
```
1. Attempts to either AutoVersion or take a manual version  
2. Gets current boto session  
3. Connect to kms  
4. Create an instance of KeyService with the kms connection, kms_key (default is alias/credstash unless overridden by -k flag), and a context (which can be empty)  
5. Call seal_aes_ctr_legacy with the instance created above, the secret inputed, and digest (DEFAULT_DIGEST by default)  
6. Calls generate_key_data with 64 bit key _calls [AWS.KMS.generate-data-key](http://docs.aws.amazon.com/cli/latest/reference/kms/generate-data-key.html) in the background_    
        * calls with _alias/credstash_ and gets a CiphertextBlock and Plaintext _blob_  
        * this is essentially a key and encoded_key  
7. Call to internal _seal_aes_ctr with the key returned from above, the secret (as plaintext), LEGACY_NONCE _a binary pad_, and the digest method (again DEFAULT_DIGEST)  
8. take the key from above and split it into a **data_key** and a **hmac_key**  
9. Create CipherInstance with _AES(data_key)_, _CTR(LEGACY_NONCE)_ mode, and the default pycrptopgrahy backend  
10. Generates an encryptor from the CipherInstance and calls encryptor.update with the secret (plaintexted and encoded) and then is locked with finalize()  
11. Calls _get_hmac with the hmac key generate from step 7, the ciphertext created from step 9, and the DEFAULT_DIGEST  
12. Generates an HMAC and updates it with the ciphertext created from step 9 and is locked with finalize()  
13. _seal_aes_ctr returns with the ciphertext and hmac  
14. seal_aes_ctr_legacy returns with a formatted object that consists of key (b64enocded encoded key from step 5), contents (b64encoded ciphertext  from step 12), hmac (hexencoded hmac from step 12), and digest (DEFAULT_DIGEST)  
15. Updates the dictionary from step 13 with name (of the secret's key) and version (that's padded)  
16. Writes to DynamoDB table  

### credstash put *TLDR*

#### credstash put *Diagram*

## credstash get
```
def getSecret(name, version="", region=None,
              table="credential-store", context=None,
              **kwargs):
```

### credstash get *TLDR*

#### credstash get *Diagram*
