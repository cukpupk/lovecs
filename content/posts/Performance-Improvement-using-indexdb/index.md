---
title: "Performance Improvement using indexdb"
date: 2022-03-05T08:06:25+06:00
description: Using IndexDb cache images
menu:
  sidebar:
    name: Markdown Sample
    identifier: markdown
    weight: 30
author:
  name: Shiwen
math: true
---
Nowadays, image sizes are getting larger while user experience is one of the critical factor evaluating the website. To ensure good user experience, performance enhancement toward images require engineer to do deep improvement. Here I am going to introduce method using indexDb to store large images and the website will not need to fetch the image once reload or reenter into the page. This work on when these images are viewed on high frequency and seldom changed.  
---

> Most of the time. To use indexDb needs complicated api according to MDN. However, there are some third-party library that can work through. Here I am introducing dexie 
---
https://dexie.org/

---
Yarn

> yarn add dexie
---
> yarn add dexie-react-hooks

---
NPM
> npm install dexie
---
> npm install dexie-react-hooks
  
---
then
> Create a file db.js (or db.ts)
## Code Blocks

#### Code block with backticks

```
import Dexie from 'dexie'

export const db = new Dexie('myDatabase')

db.version(1).stores({
  projectImages: '++id, name, url, objectUrl'
})
```
#### import db file and use it after getting response from back end. We are going to connect to the DB and store the images into the DB. But first of all, what we have is a string url, we need to fetch the image from url and convert to base64

     export const toDataURL = url => fetch(url)
         .then(response => response.blob())
         .then(blob => new Promise((resolve, reject) => {
           const reader = new FileReader()
           reader.onloadend = () => resolve(reader.result)
           reader.onerror = reject
           reader.readAsDataURL(blob)
         }))

#### Then we are going to store the url and the base64 image into db. We are going to use url as our key to compare the image from response and image at local db. If urls are different, we are going to refetch the image and cache it at db, otherwise we use base64 image at db.
{{< highlight html >}}
 let results = response.results //response from backend. Response is an object arr contains images info
for (const item of results) {
        try {
          const imageUrlArr = await db.projectImages.where("url").equalsIgnoreCase(item.logo.imgUri).toArray()
          if(imageUrlArr.length > 0) {
            item.objectUrl = imageUrlArr[0].objectUrl
          } else {
            if(item.logo.imgUri !== '' && item.logo.imgUri !== null) {
              toDataURL(item.logo.imgUri).then(dataUrl => {
                db.projectImages.add({
                  name: item.name,
                  url: item.logo.imgUri,
                  objectUrl: dataUrl
                })
              })
            }
          }
        } catch (error) {
          console.log(error)
          db.projectImages.clear()
        }
      }
{{< /highlight >}}

> Now we implement the functionality that cache the images on local and do not need to fetch each image every time we get the response from backend. Images will immediately display after we get the response. User experience enhanced!

## We are not finished yet. Although indexdb is very large compare to 5mb limit size of localStorage and sessionStorage. We need to check whether the storage has enough space to cache. Different Browser has different strategies toward indexDb. 
The maximum browser storage space is dynamic — it is based on your hard drive size. The global limit is calculated as 50% of free disk space. In Firefox, an internal browser tool called the Quota Manager keeps track of how much disk space each origin is using up, and deletes data if necessary.

So if the free space on your hard drive is 500 GB, then the total storage for a browser is 250 GB. If this is exceeded, a process called origin eviction comes into play, deleting an entire origin's worth of data until the storage amount goes under the limit again. There is no trimming effect put in place to delete parts of origins — deleting one database of an origin could cause problems with inconsistency.

There's also another limit called group limit — this is defined as 20% of the global limit, but it has a minimum of 10 MB and a maximum of 2 GB.

## We use StorageEstimate API to detect the rest of the storage size. And Clean the db by selecting most unused images every time the storage size hits the bar. I set 20% of the total size here.
> Chrome allow browser to use at most 80% disk size。One single source can use up to 60% of disk.You can use StorageManager API to check the max usable size.
---
> Internet Explorer 10 or higher version can store up to 250 MB, and will warn user when use more than 10 MB
---
> Firefox allow browser to use at most 50% disk size. eTLD+1 group（like example.com;www.example.com ; foo.bar.example.com）use up to 2 GB. You can use StorageManager API to check the max usable size.
---
> Safari（desktop and mobile）allow up to 1 GB. When it hits the limit, Safari will warn user and increase the storage size once by 200 MB.
---
{{< highlight html >}}
if (navigator.storage && navigator.storage.estimate) {
  const quota = await navigator.storage.estimate();
  // quota.usage -> Bytes used
  // quota.quota -> Max Bytes can be used
  const percentageUsed = (quota.usage / quota.quota) * 100;
  console.log(`You have used ${percentageUsed}%。`);
  const remaining = quota.quota - quota.usage;
  console.log(`You can write ${remaining} Bytes more`);
}
{{< /highlight >}}