## Adding an image control to store the image

> In Html file
``` html
<input type = "file" #filePicker (change)="onPickedImage($event)">
```

> In typescript file
``` typescript
onPickedImage(event:Event){
const file = (event.target as HTMLInputElement).files[0];
this.form.patchValue({image:file}) //patchValue allow us to target a single control
this.form.get('image').updateValueAndValidity();
}
```

## Adding an image preview
> In Html file 
``` html
<div *ngIf="imagePreview != '' && imagePreview" && form.get('image').valid>
  <image [src]="imagePreview" alt="form.value.title">
</div>
```

> In typescript
``` typescript
imagePreview: string;

onPickedImage(event:Event){
const file = (event.target as HTMLInputElement).files[0];
this.form.patchValue({image:file}) //patchValue allow us to target a single control
this.form.get('image').updateValueAndValidity();
const reader = new FileReader();
reader.onload = () => {
  this.imagePreview = reader.result;
  //typescript might give an error here, use "reader.result as string" instead of "reader.result".
};
reader.readAsDataUrl(file);
}
```
## Mime-type validator
Now, lets validate our file which should be of valid image type. There is no buildin validators, so we have to write our own one.
let's do it.
> At mime-type.validators.ts
``` typescript

   export const mimeType = (control:AbstractControl):Promise<{[key:string]:any}> | Observable<{[key:string]:any}> =>{
     const file = control.value as File;
     const fileReader = new FileReader();

     //creating a custom observable
     const frObs = Observable.create((observer:Observer < { [key:string]:any }>)=>{
         fileReader.addEventListener("loadend",()=>{         // equivalent to fileReader.onloadend()
            const arr  = new Uint8Array(fileReader.result as ArrayBuffer).subarray(0,4) //creating the array of 8 bit
            //typescript might give you an error here. Use "fileReader.result as ArrayBuffer" instead of just "fileReader.result 
            let header = "";
            let isValid = false;
           for(let i=0;i<arr.length;i++){
               header+= arr[i].toString(16);
            }

            switch (header) {
                case "89504e47":
                    isValid = true;
                    break;
                case "ffd8ffe0":
                case "ffd8ffe1":
                case "ffd8ffe2":
                case "ffd8ffe3":
                case "ffd8ffe8":
                    isValid = true;
                    break;
                default:
                    isValid = false; // Or you can use the blob.type as fallback
                    break;
            }

            if(isValid){
                observer.next(null)
            }else{
                observer.next({invaildMimeType:true})
            }
            observer.complete();
        });
        fileReader.readAsArrayBuffer(file);
     });
      return frObs;
     };
```
> To use mime-type validators import mimeType from mime-type.validators.ts file and simply add it to image control as asyncValidators
``` typescript
import {mimeType} from './mime-type-validators';
import {FormGroup,FormControl} from '@angular/forms';
form:FormGroup;
ngOninit(){
this.form = new FormGroup({
image: new FormControl(null,{asyncValidators:[mimeType]})
})
}

```

## Server Side Upload
We can't extract an image on server with body-parser,so we neeed to install different package called multer.

**Installation**<br>
``` yarn add multer ```<br>
**Implementation**
``` javascript
const multer = require('multer');
const express = require('express');

const router = express.Router();

//mime type map 
const MIME_TYPE_MAP = {
    'image/png': 'png',
    'image/jpeg': 'jpg',
    'image/jpg': 'jpg'
};

//multer storage
const storage  = multer.diskStorage({
   destination:(req,file,cb) =>{
       const isValid = MIME_TYPE_MAP[file];
        let error = new Error("Invalid mime type");
        if(isValid){
            error = null;
        }
       cb(error, "backend/images");
   },

   filename:(req, file, cb) => {
       const name = file.originalname.toLowerCase().split(' ').join('-')
       const ext= MIME_TYPE_MAP[file.mimetype];
       cb(null, name + '-' + Date.now() + '.' + ext );
   }
});

router.post('',multer({storage:storage}).single('image),(req,res,next)=>{
})
```
## Uploading files
JSON can't include a file, so instead of sending json as a body to the server, i will now send a form data. Form data is basically a data format which allows us to combine a text value and file values. Now, let's implement it.
``` typescript
   saveData(title:string,content:string,image:File){
   const postData = new FormData();
   postData.append('title',title);
   postData.append('content',content);
   postData.append('image',image);
   this.http.post('http://localhost:3000/api/create',postData)
   }
```

## Working with the file url
``` javascript
router.post('',multer({storage:storage}).single(req,res, next)=>{
  const url = req.protocol + '://' + req.get("host");
  const post = new Post({
  title:req.body.title,
  content:req.body.content,
  imagePath:url + '/images/'
  })
})
```
