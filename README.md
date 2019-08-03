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

