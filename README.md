# File Uploading & Display - Fullstack

Find here summarized a mini guide how to incorporate file uplading with a (free) file cloud provider.

We will use Cloudinary as the provider, which allows us (at the time of writing) 10GB+ storage for free.

We will use the example of an Avatar upload.

## Steps 

* Signup to Cloudinary: https://cloudinary.com/users/register/free
* Free Plan & Pricing: https://cloudinary.com/pricing

### Backend part

* Install following packages: `npm i multer datauri cloudinary dotenv` 
* Configure Cloudinary URL in a .env of your backend

* Load the .env in your server.js file (if not done already):  `require("dotenv").config() `

* Setup an upload middleware with Multer and export it:
```
  const multer = require("multer")
  const upload = multer({ storage: multer.memoryStorage() })
  module.exports = upload
```

* Adapt your model where you wanna attach an image URL
  * Example User Model: add a field "avatar_url" (String)

* Attach middleware to a route which should receives uploads
  * e.g. in users route `router.post('/', upload.single('avatar'), (req, res, next) => {}`
  * It is important to tell multer the exact FIELD NAME you used in the form (!) in upload.single("FIELD_NAME")
  * e.g. Form input was defined like this `<input type="file" name="myImage" />`
  * Multer parsing: `upload.single("myImage"")`
  * Multer will make the parsed file in a special object `req.file`

* Perform upload
  * Convert the received binary data (stored in req.file) into a DataURI (base64 encoded file string):
    * https://www.npmjs.com/package/datauri#from-a-buffer
  * Upload the string to cloudinary
    * `const result = await cloudinary.upload.upload( dataUriParseResult.content )`
  * Store the received URL in your model
    * e.g. ` const userNew = await User({ ...req.body, avatar_url: result.secure_url })
  * Return the created user to the frontend using res.json()

* Test File upload against your route from Insomnia
  * Select as BODY type "multipart / form data"
    * Now you can set key value pairs
  * You can now select also "file" as field type and choose a file from your filesystem

### Frontend part

* Create or use an existing React Frontend to upload a file
  * Ideally some app with a signup or any other upload form

* Notes on usage with React-Hook-Form
  * Once you register an input by putting the "register" key on it, you cannot put an additional "onChange" handler on it anymore
  * But in order to listen for the Avatar File Selection (and show a preview) we need to listen for this
  * The simplest way is to simply not handle the file input by React-Hook-Form
    * `<input name="avatar" type='file' accept='image/*' onChange={onAvatarChange} /> ` 

* Submitting mixed data
  * When we want to submit mixed data (so called "multipart form data") we need to send the data to the API differently
  * In Axios we need to prevent the default Content-Type "application/json"
    * ` axios.post('/myApiUploadUrl, data, { headers: { 'Content-Type': 'undefined' } })`
    * Now we are ready to send mixed data to the backend

* Submitting
  * If we wanna send mixed data, we cannot simply pass in JSON data anymore to Axios
  * We need to pass in a FormData object
  * Example:
    ```
      onSubmit = (data) => {
        const formData = new FormData()

        // attach avatar (stored in state)
        formData.append('avatar', avatarFile)

        // loop through form data and append
        for(let key in data) {
          formData.append(key, data[key])
        }

        axios.post('/myApiUploadUrl', formData, { headers: { 'Content-Type': 'undefined' }})
      }
    ```

### BONUS: Deploy backend & frontend

  * Deploy the backend (do not forget to outsource all sensitive stuff to .env)
  * Set each prod env variables on heroku using `heroku config:set KEY=VALUE`
  * Test your deployed backend with your frontend running on localhost first
  * Once working: Deploy your frontend to vercel
  * Put the generated Frontend URL / Domain into FRONTEND_ORIGIN with heroku config:set


## Resources

- Multer Documentation: https://github.com/expressjs/multer 
- Cloudinary package: https://www.npmjs.com/package/cloudinary
- Uploading a base64 encoded image (string) with Node: https://support.cloudinary.com/hc/en-us/community/posts/360009192212-Uploading-image-from-a-base64-string
- Data URI package for converting binary file (buffer) into dataURI: https://www.npmjs.com/package/datauri#from-a-buffer
- MongoDB ATLAS storage limits: https://www.mongodb.com/pricing
- MongoDB ATLAS - what will happen on reaching storage limit? https://docs.atlas.mongodb.com/reference/faq/storage/