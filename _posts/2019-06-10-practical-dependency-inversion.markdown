---
layout: post
title:  "Dependency inversion: practical Node.js example"
date:   2019-06-10 22:49:27 +0200
categories: engineering
tags: [design-patterns, solid, nodejs]
---

## Introduction

[Dependency inversion](https://en.wikipedia.org/wiki/Dependency_inversion_principle) is one in a set of well-known SOLID design principles. It states that:

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

It might not be easy to find a relevant practical case to get a grasp of this, especially when using languages like JavaScript that does not have Interface concept. This article is my humble attempt to provide an illustrated example of how understanding of this principle could make code of a typical Node.js web application slightly better.

## Pre-problem state

Let's consider an app that, along other tasks, performs actions on AWS S3 storage.

We have a Recording mongoose schema that contains a `filePath` string, which refers to a path on S3:
```js
const mongoose = require('mongoose');
const RecordingSchema = new mongoose.Schema(
  {
    filePath: { type: String, required: true }
  }
);
```

In one of use cases it should be possible to download file stored on S3 from user's browser, which requires creating a signed URL. That led to creation of `getUrl` convenience method within the same model file:

```js
const S3 = require('aws-sdk/clients/s3');

const s3Client = new S3({
  accessKeyId: process.env.AWS_ACCESS_KEY_ID,
  secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
});

RecordingSchema.methods.getUrl = function() {
  return s3Client.getSignedUrl('getObject', {
    Key: this.filePath,
    Bucket: 'myRecordingsBucket'
  });
};
```

In the other case user should be able to upload a file and server should store it to S3. This requirement led, in turn, to creation of `persist` method:

```js
RecordingSchema.methods.persist = function(data) {
  return s3Client.putObject({
    Bucket: 'myRecordingsBucket',
    Key: this.filePath,
    Body: data
  });
};
```

## Problem state

Now it's when a problem begins. Let's say each time when user uploads a file for Recording, we want to store it to a backup bucket called `myRecordingsBackupBucket`, having a key postfixed with `(backup)`. I.e. if a recording has `filePath: "my-recording-1"`, it should be also stored as `myRecordingsBackupBucket/my-recording-1(backup)`.

How could we approach this? We could add `persistBackup` method:

```js
RecordingSchema.methods.persistBackup = function(data) {
  return s3Client.putObject({
    Bucket: 'myRecordingsBackupBucket',
    Key: `${this.filePath}(backup)`,
    Body: data
  });
};
```

and end up having two separate methods for doing essentially similar operation, or parameterize `persist` method:

```js
RecordingSchema.methods.persist = function(isForBackup = false, data) {
  return s3Client.putObject({
    Bucket: isForBackup ?
      'myRecordingsBackupBucket' :
      'myRecordingsBucket',
    Key: isForBackup ?
      `${this.filePath}(backup)` :
      this.filePath,
    Body: data
  });
};
```

Based on my experience, having flags like `isForBackup` in the code to branch off the logic might be a clear indicator of dependency inversion principle violation.

## Analysis

What exactly is wrong with the code above and what can we do to make it better?

First of all, any potential consumer of `persist` method resides closer to input (request handler in this case) and so is a lower-level module than `RecordingSchema` is.

Second observation - when we directly require S3 client constructor and create an instance of it on our schema file definition, we make the latter depend on the former. What's bad about it? Several things in fact. Writing isolated unit tests for such model file becomes much trickier - and we most likely end up mocking `aws-sdk/clients/s3` constructor. Second point is flexibility - one day we need to switch from S3 to some other storage, and that will require rewriting both `persist` and `getUrl` methods to make use of a new client, which can introduce bugs.

## Inverting dependency

Applying dependency inversion principle to this case, we can formulate: `RecordingSchema` should not depend on S3 client. Both should depend on abstraction.

What kind of abstraction we can think of? Essentially there are two operations to be performed - getting URL for a particular `filePath` from a storage and uploading data into storage for particular `filePath`. As we refer to `storage` twice in te statement above, using it as an abstract entity seems justified.

Unfortunately, JavaScript does not have a concept of interface, so we have no option to represent it in code. Using TypeScript we could come up with something like the following:

```ts
interface Storage {
  load: (key: string) => Promise<Buffer>,
  save: (key: string, data: Buffer) => Promise<void>
}
```

and make our `RecordingSchema` to rely on this interface for doing its job:

```ts
RecordingSchema.methods.getUrl = function(storage: Storage) {
  return storage.load(this.filePath);
};

RecordingSchema.methods.persist = function(storage: Storage, data: Buffer) {
  return storage.save(this.filePath, data);
};
```

Implementing it in plain JavaScript won't change anything expect omitting type annotations and keeping `Storage` interface in your head rather than in code.

Note that such change allows us to drop require of `aws-sdk/clients/s3` entirely, so no more mocking in unit tests is required - just implement some mock storage that would satisfy `Storage` (using [Jest](https://jestjs.io) as testing framework in this example):

```js
// somewhere in Recording model test
const mockStorage = {
  load: jest.fn().mockReturnValue(Promise.resolve(Buffer.from('fake'))),
  save: jest.fn().mockReturnValue(Promise.resolve())
};

const someFakeData = Buffer.from('fake');

await recording.persist(mockStorage, someData)

expect(mockStorage.save).toBeCalledWith(recording.filePath, someFakeData);
```

The last thing to decide is when to create a concrete instance of `Storage`. Personally I tend to do that at the lowest possible level. For a web server for example, I'd do it when the app is reading its configuration settings and server gets initialized. Such approach allows you to maximize number of further usage options.

Whatever particular use case is, it seems like a good idea to create helper modules that produce concrete instances and deal with details, e.g. `makeS3Storage.js`:
```js
const S3 = require('aws-sdk/clients/s3');

const s3Client = new S3({
  accessKeyId: process.env.AWS_ACCESS_KEY_ID,
  secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
});

module.exports = function makeS3Storage(bucketName) {
  return {
    load: (key) => s3Client.getSignedUrl('getObject', {
      Key: key,
      Bucket: bucketName
    }),

    save: (key, data) => s3Client.putObject({
      Bucket: bucketName,
      Key: key,
      Body: data
    });
  };
}
```

## Conclusion
In this article we went through a typical example of dependency inversion principle violation and (hopefully) made the code a bit better by decreasing coupling between the modules and making it more testable.
