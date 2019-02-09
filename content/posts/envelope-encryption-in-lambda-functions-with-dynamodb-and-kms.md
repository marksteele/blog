---
layout: blog
title: Envelope encryption in Lambda functions with DynamoDB and KMS
date: '2018-05-03T10:44:37-04:00'
cover: /images/envelope-encryption.png
categories:
  - Security
  - AWS
  - Lambda
  - Encryption
  - DynamoDB
  - KMS
  - serverless
---
![](/images/envelope-encryption.png)

Here's a quick code snippet on how to implement field level encryption of data stored in DynamoDB using per-record encryption keys and the AWS Key management store (KMS).

<!--more-->

Before starting you'll want to create a KMS key that will just be used for this service and take note of the Key ID.

Also, I recommend enabling at-rest encryption of the DynamoDB table. 

This example assumes a table structured similar to this:

```json
{
  id: 'foobar',
  nonSensitiveStuff: 'will not be encrypted',
  sensitiveStuff: 'will be encrypted',
  otherStuff: 'will not be encrypted'
}
```

the `id` field is the primary key. As an enhancement, if your primary key is also sensitive you could instead use a hash of the ID instead of the actual value (although you'd lose the ability to search through the IDs).

``` javascript 
const AWS = require('aws-sdk'); 
const crypto = require('crypto');

const ALGO = 'aes256';

class EnvelopeEncrypt {

  // Asuming input is object, and field we want to encrypt is sensitiveStuff
  // Also assuming that there is an 'id' field which is the primary key.
  // And also this assume UTF8 content...
  saveData(userInfo) { 
    const kms = new AWS.KMS();
    return kms.generateDataKey({ KeyId: 'YOUR_KMS_ENCRYPTION_KEY_ID', KeySpec: 'AES_256' }).promise()
      .then((dataKey) => {
        Object.assign(userInfo,{
          dataKey: dataKey.CiphertextBlob,
          sensitiveStuff: this.encrypt(dataKey.Plaintext, userInfo.sensitiveStuff)
        });
        const params = {
          TableName: 'YOUR_DYNAMO_DB_TABLE_NAME',
          Item: userInfo,
        };
        const dynamodb = new AWS.DynamoDB.DocumentClient();
        return dynamodb.put(params).promise();
      });
  }

  loadData(id) {
    const dynamodb = new AWS.DynamoDB.DocumentClient();
    const params = {
      TableName: 'YOUR_DYNAMO_DB_TABLE_NAME',
      Key: { id },
    };
    return dynamodb.get(params).promise()
      .then((record) => {
        if (record.Item !== undefined) {
          const kms = new AWS.KMS();
          return kms.decrypt({ CiphertextBlob: Buffer.from(record.Item.dataKey, 'base64') }).promise()
            .then((dataKey) => {
              Object.assign(record.Item, { sensitiveStuff: this.decrypt(dataKey.Plaintext.toString('base64'), record.Item.sensitiveStuff) });
              return record.Item;
            });
        }
        return undefined;
      });
  }

  encrypt(keyBase64, payload) {
    const cipher = crypto.createCipher(ALGO, Buffer.from(keyBase64, 'base64'));
    return (Buffer.concat([cipher.update(payload, 'utf8'), cipher.final()])).toString('base64');
  }

  decrypt(keyBase64, payload) {
    const cipher = crypto.createDecipher(ALGO, Buffer.from(keyBase64, 'base64'));
    return (Buffer.concat([cipher.update(payload, 'base64'), cipher.final()])).toString('utf8');
  }
}

module.exports = EnvelopeEncrypt;
```