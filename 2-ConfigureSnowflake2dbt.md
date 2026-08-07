# Connecting dbt to a Snowflake database

As Snowflake has DEPRECATED the use of single-factor password sign ins we are dependent on using safer methods of congigurating accounts to connections. 

### Key Pair Authentication 

Step 1 - Encrypt a private key 

(Option A: with passphrase - more secure)

~~~~bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 des3 -out rsa_private_key.p8
~~~~

(Option B: withouth passpharese )

~~~~bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -nocrypt -out rsa_private_key.p8
~~~~

Then we generate a publick key from the private key 

~~~~bash 
openssl rsa -in rsa_private_key.p8 -pubout -out rsa_public_key.pub
~~~~


The Snowflake user account is configured with the private key 

~~~~sql
ALTER USER <username> SET RSA_PUBLIC_KEY='<public_key_string>';
~~~~


The private key is stored securely on dbt to allow the handshake. 

---

If using option A. On the terminal (with ```OpenSSL``` installed) it will request a passkey ensure to store this. 

Once produced we should have an a PRIVATE KEY key ```rsa_private_key.p8```  and a public key ```rsa_public_key.pub``` we can use ```cat``` command to review it. 


---

### dbt Target Schema 

The target schema is the schema that the tables created in dbt will fall in. Try to make this a unique schema so it doesn't ovewrite (if in Production)

Managed Repository 

Is managed by dbt and all the code is stored. Will store the code. 





