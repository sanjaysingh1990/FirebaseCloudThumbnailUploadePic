# FirebaseCloudThumbnailUploadePic
### create a thumbnail of uploaded pic in size [300, 400,450]
```
const functions = require('firebase-functions');
const {Storage} = require("@google-cloud/storage");
const spawn = require('child-process-promise').spawn;
const mkdirp = require('mkdirp-promise');
const path = require('path');
const fs = require('fs-extra');
const sharp = require('sharp');
const { tmpdir }= require('os');
const { join, dirname }=  require('path');

exports.optimizeImages= functions.storage
  .object()
  .onFinalize(async object => {
          // Creates a client
            const storage = new Storage({
              projectId: 'vntspush',
            });

    const bucket = storage.bucket(object.bucket);
    const filePath = object.name;
    const fileName = filePath.split('/').pop();
    const bucketDir = dirname(filePath);
    const workingDir = join(tmpdir(), 'thumbs');
    const tmpFilePath = join(workingDir, 'source.png');


    if (fileName.includes('thumb@') || !object.contentType.includes('image')) {
      console.log('exiting function');
      return false;
    }

    // 1. Ensure thumbnail dir exists
    await fs.ensureDir(workingDir);
    // 2. Download Source File
    await bucket.file(filePath).download({
      destination: tmpFilePath
    });

    // 3. Resize the images and define an array of upload promises
    const sizes = [300, 400,450];

    const uploadPromises = sizes.map(async size => {
      const thumbName = 'thumb@'+size+'_'+fileName;
      const thumbPath = join(workingDir, thumbName);

      // Resize source image
      await sharp(tmpFilePath)
        .resize(size, size)
        .toFile(thumbPath);

      // Upload to GCS
      return bucket.upload(thumbPath, {
        destination: join(bucketDir, thumbName)
      });
    });

    // 4. Run the upload operations
    await Promise.all(uploadPromises);

    // 5. Cleanup remove the tmp/thumbs from the filesystem
    return fs.remove(workingDir);
  });


```

