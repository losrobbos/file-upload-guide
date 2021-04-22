# File Uploading & Display - Fullstack

Find here summarized a mini guide how to incorporate file uplading from frontend to backend with a (free) file cloud provider.

We will use Cloudinary as the provider, which allows us (at the time of writing) 10GB+ storage for free.

We will use the example of an Avatar upload.

## Steps 

* Signup to Cloudinary: https://cloudinary.com/users/register/free
* Free Plan & Pricing: https://cloudinary.com/pricing

### Backend part

* Install following packages: `npm i multer datauri cloudinary dotenv` 
* Configure Cloudinary URL in a .env of your backend
  * The Cloudinary URL is similar to your MongoDB Atlas URL. A url to your own cloud including your user credentials
  * You find the Cloudinary Environment URL in the cloudinary dashboard after login

* Load the .env in your server.js file (if not done already):  `require("dotenv").config() `

* Setup an upload middleware with Multer and export it:
```
  const multer = require("multer")
  const upload = multer({ storage: multer.memoryStorage() })
  module.exports = upload
```

* Adapt your model where you wanna attach an image URL
  * Example User Model: add a field "avatar_url" (String)

* Attach middleware to a route which should receive uploads
  * e.g. in users route `router.post('/', upload.single('avatar'), (req, res, next) => {}`
  * It is important to tell multer the exact FIELD NAME you used in the form (!) in upload.single("FIELD_NAME")
  * e.g. Form input was defined like this `<input type="file" name="myImage" />`
  * Multer parsing: `upload.single("myImage"")`
  * Multer will put the parsed file data into a special object `req.file`

* Perform upload
  * Convert the received binary data (stored in req.file) into a DataURI (base64 encoded file string):
    * https://www.npmjs.com/package/datauri#from-a-buffer
  * Upload the string to cloudinary
    * `const result = await cloudinary.upload.upload( dataUriParseResult.content )`
  * Store the received URL in your model
    * e.g. `const userNew = await User({ ...req.body, avatar_url: result.secure_url }) `
    * Cloudinary will provide you with TWO urls in its response: url and secure_url
     * url will be reachable by http:// and secure_url will be reachable by https://. So it is advisable to always use the secure one 
  * Return the created user to the frontend using res.json()

* Test File upload against your route from Insomnia
  * Select as BODY type "multipart / form data"
    * Now you can set key value pairs
  * You can now select also "file" as field type and choose a file from your filesystem

### Frontend part

* Create or use an existing React Frontend to upload a file
  * Ideally some app with a signup or any other upload form

* Create an input of type "file", e.g.
  * `<input name="avatar" type='file' accept='image/*' onChange={onAvatarChange} /> `
  * This will make it possible to select files from your filesystem

* Handle the avatar selection
  * Setup a state for storing the avatar file: `const [avatarFile, setAvatarFile] =  useState()`
  * Define an onAvatarChange handler `const onAvatarChange = (e) => {...}`
  * The file, the user selected, will be availble in the event object: `e.target.files[0]`
  * If you allowed multiple file selection (with `<input type="file" multiple />`) you will have an array of files in `e.target.files`
  * Store the avatar file in state `setAvatarFile( e.target.files[0] )`

* Previewing an image
  * In case you wanna have a preview of the selected file, you can generate a "BLOB" Url (BLOB = Binary Large Object)
  * Setup a state for storing this avatar preview url: `const [avatarPreview, setAvatarPreview] =  useState( )`
  * Use the builtin browser method `URL.createObjectUrl()` to generate the blob url and store it in state: 
   ```
   const fileUrl = URL.createObjectURL( e.target.files[0] )
   setAvatarPreview( fileUrl )   
   ```
   * ... and assign it to an image tag for displaying the preview `<img src={ avatarPreview } />`
  * Et voila: Now you have an avatar preview on file selection
  * To select a file on image click, you can use the label trick 
    * Put an id on the HTML file input (e.g. type="file" id="avatar") 
    * Wrap your preview image with a label and link it to the input `<label htmlFor="avatar"><img src={ avatarPreview } /></label>`
    * Now you can hide the ugly default file input field, e.g. with simple CSS (visibility: hidden)

* Notes on usage with React-Hook-Form
  * Once you register an input by putting the "register" key on it, you cannot put an additional "onChange" handler on it anymore
  * But in order to listen for the Avatar File Selection (and show a preview) we need that onChange handler...
  * The simplest way is to simply not handle the file input by React-Hook-Form

* Submitting mixed data
  * When we want to submit mixed data (so called "multipart form data") we need to send the data to the API differently
  * In Axios we need to prevent the default Content-Type "application/json"
    * `axios.post('/myApiUploadUrl, data, { headers: { 'Content-Type': 'undefined' } })`
    * The browser will automatically convert this Content-Type into "multipart/form-data" + necessary configuration in order to make the mixed upload work
    * Now we are ready to send mixed data to the backend

* Submitting
  * If we wanna send mixed data, we cannot simply pass in JSON data anymore to Axios
  * We need to pass in a FormData object
  * Example:
    ```
      onSubmit = (data) => {
        const formData = new FormData()

        // attach avatar binary file (stored in state)
        formData.append('avatar', avatarFile)

        // loop through JSON data and append
        for(let key in data) {
          formData.append(key, data[key])
        }

        axios.post('/myApiUploadUrl', formData, { headers: { 'Content-Type': 'undefined' }})
      }
    ```

### Uploading multiple files

Let's assume in a form you wanna upload images for a recipe of your favorite food.

#### Frontend

Simply add the "multiple" HTML attribute to your input field 

` <input type="file" name="recipe_images" accept="image/*" multiple />`

#### Backend

Now instead of using "upload.single" we need to use the "upload.array" method of Multer:

` router.post("/recipe", upload.array("recipe_images"), ...yourController...) `

Important: Instead of req.file the contents will now get stored in the variable `req.files`. This will be an array of file objects.

Each object in the array req.files has all the file properties, including "originalname" (for the uploaded filename) + the field "buffer" with the binary image data.


### Uploading two different file fields

Let's assume you wanna manage your user avatar + your family photos in your profile.

Now we will have two DIFFERENT file input fields we wanna upload. How do we do this?

##### Frontend Form

For the avatar we can set an input to select a single file.
For the family photos we set an input field for selecting multiple files.

```
<input type="file" name="avatar" accept="image/*" />
<input type="file" name="family" accept="image/*" multiple />
```

##### Backend

To tell Multer to scan for files in several received file inputs, we can use the `upload.fields()` method, which expects an array of form field names.

```
` router.post("/recipe", upload.fields([{name: "avatar"}, { name: "family", maxCount: 3 }]), ...yourController...) `

```
Again, all your files will now be available in `req.files`.

How to now distinguish the avatar image from the family images in the req.files array?

Well, each file object in the array has the field "fieldname".

E.g.: 
```
{ 
  originalname: 'my_avatar.png',
  buffer: <...SomeLenghtyBinaryThing...>
  fieldname: "avatar"
}

```

So by checking the fieldname, you will know which type of image was uploaded.


## Resources

- Multer Documentation: https://github.com/expressjs/multer 
- Cloudinary package: https://www.npmjs.com/package/cloudinary
- Uploading a base64 encoded image (string) with Node: https://support.cloudinary.com/hc/en-us/community/posts/360009192212-Uploading-image-from-a-base64-string
- Data URI package for converting binary file (buffer) into dataURI: https://www.npmjs.com/package/datauri#from-a-buffer
- MongoDB ATLAS storage limits: https://www.mongodb.com/pricing
- MongoDB ATLAS - what will happen on reaching storage limit? https://docs.atlas.mongodb.com/reference/faq/storage/
