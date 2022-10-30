# File Uploading & Display - Fullstack

Find here summarized a mini guide how to incorporate file uplading from frontend to backend with a (free) file cloud provider.

We will use Cloudinary as the provider, which allows us (at the time of writing) 10GB+ storage for free.

To see an minimal fullstack code example of this guide, see this demo: https://github.com/losrobbos/file-string-upload-fullstack

We will use the example of an Avatar upload.

In this fullstack sample we will encode our images as base64 strings BEFORE uploading them to the API. So we will do the conversion in the frontend.

This way we do not need to send "multipart formdata" and just upload a JSON object as usual, which simplifies the overall workflow quite a bit.

Also in the backend we then do not need to parse binary data anymore. We can simply upload directly the received base64 string to our file cloud provider. This way we can completely skip classical file parsing middleware like Multer.

## Steps 

* Signup to Cloudinary: https://cloudinary.com/users/register/free
* Free Plan & Pricing: https://cloudinary.com/pricing

### Backend part

* Install following packages: `npm i cloudinary dotenv` 
* Configure Cloudinary URL in a .env of your backend
  * The Cloudinary URL is similar to your MongoDB Atlas URL. A url to your own cloud including your user credentials
  * You find the Cloudinary Environment URL in the cloudinary dashboard after login
   * Just pick the variable "CLOUDINARY_URL" from the dashboard and place it in your .env

* Load the .env at the top of your server.js file (if not done already):  `require("dotenv").config() `

* Recommended: Adapt the JSON upload limit in your JSON body parser middleware. The default upload limit is 100KB!
  * So in order to allow e.g. uploads of 1MB of data, state: `app.use( express.json({ limit: '1MB' })`

* Adapt your model where you wanna store an image URL
  * Example User Model: add a field "avatar_url" (String)

* Upload route handler (=controller)
  * Import cloudinary at the top of your file: `const cloudinary = require("cloudinary").v2`   
  * Extract / split the avatar file and normal JSON data from req.body: `const { avatar, ...userData } = req.body `
  * Upload the avatar string to cloudinary
    * `const result = await cloudinary.upload.upload( avatar )`
  * Store the received URL in your model
    * e.g. `const userNew = await User.create({ ...userData, avatar_url: result.secure_url }) `
    * Cloudinary will provide you with TWO urls in its response: url and secure_url
     * url will be reachable by http:// and secure_url will be reachable by https://. So it is advisable to always use the secure one 
  * Return the created user to the frontend using res.json()

* Test File upload against your route with your favorite REST client
  * Setup a POST request to your API upload URL
  * Use JSON as Body format
  * Convert some avatar file to a base64 dataUri string, e.g. here: https://www.base64-image.de/ and copy the generated string into the clipboard
  * Paste the dataUri string into your JSON Body "avatar" field
  * Example body:
  ```
  {
     "email": "rob@rob.com",
     "avatar: "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAo....."
  }
  ```

### Frontend part

* Create or use an existing React Frontend to upload a file
  * Ideally some app with a signup or any other upload form

* Create an input of type "file", e.g.
  * `<input name="avatar" type='file' accept='image/*' onChange={ onAvatarChange } /> `
  * This will make it possible to select files from your filesystem

* Handle the avatar selection
  * Setup a state for storing the avatar file: `const [avatarPreview, setAvatarPreview] =  useState()`
  * Define an onAvatarChange handler `const onAvatarChange = (e) => {...}`
  * The file, the user selected, will be availble in the event object: `e.target.files[0]`
  * If you allowed multiple file selection (with `<input type="file" multiple />`) you will have an array of files in `e.target.files`
  * e.target.files contains pointers to the binary files. These we now need to convert to DataURI Strings using the Browser builtin `FileReader` class
  * Example: 
    ```
    let fileSelected = e.target.files[0]  // grab selected file

    if(!fileSelected) return

    let fileReader = new FileReader()
    fileReader.readAsDataURL( fileSelected ) // concert to base64 encoded string
    // wait until file is fully loaded / converted to base64 (once fully loaded the "onloadedend" event below fires)
    fileReader.onloadend = (ev) => {
      setAvatarPreview( fileReader.result )
    }
    ```
    
* Previewing an image
  * Import some default preview image from your React folder, e.g. `import avatarDefault from './img/avatarDefault.png' ` 
  * Asign it as default to your avatar preview state: `const [avatarPreview, setAvatarPreview] =  useState( avatarDefault )`
  * Assign the state to an image tag for displaying the preview `<img src={ avatarPreview } />`
  * Et voila: Now you have an avatar preview on file selection
  * To select a file on image click, you can use the label trick 
    * Put an id on the HTML file input (e.g. type="file" id="avatar") 
    * Wrap your preview image with a label and link it to the input `<label htmlFor="avatar"><img src={ avatarPreview } /></label>`
    * Now you can hide the ugly default file input field, e.g. with simple CSS (visibility: hidden)

* Notes on usage with React-Hook-Form
  * Once you register an input by putting the "register" key on it, you cannot put an additional "onChange" handler on it anymore
  * But in order to listen for the Avatar File Selection (and show a preview) we need that onChange handler...
  * So the simplest way to avoid this, is by not handling the file input by React-Hook-Form (not putting a register key on it)

* Submitting
  * Before sending the data to the API, we need to merge the encoded avatar file into the form JSON data
  * Afterwards we can forward all data, e.g. with Axios, to the API
  * Full submit Example:
    ```
      onSubmit = (jsonData) => {
      
        jsonData.avatar = avatarPreview // merge the avatar string into our data

        // signup user with avatar in backend
        try {
          let response = await axios.post('http://localhost:5000/users', jsonData)
          // do something with the user, e.g. store in React Context....
          
          // forward to user profile
          history.push('/profile')  
        }
        // handle error
        catch(errAxios) {
          console.log(errAxios.response && errAxios.response.data)
        }
      }
    ```

## Performance

Waiting for the cloudinary response can take time. 

Especially when you deploy your page to the web. Then your frontend sends all signup data over the net to your API, your API reaches out to MongoDB ATLAS and afterwards to Cloudinary. So a huge caravan of data is going through the web :) Until all that finishes, time can accumulate to several seconds!

To increase the response to the frontend significantly you can do the following:

Send a response from your API to the frontend as fast as possible! 

How? Create the user in the database FIRST and store the base64 encoded string temporarily as your avatar_url.

You can use the same field in the DB schema. An encoded file is a valid "uri", that's why you can put it in the "src" attribute of the `<img>` tag easily.

Once you have stored the user in the database -> send the response back immediately. And upload now the avatar AFTERWARDS to cloudinary! 

So the process would then look like this in your signup route:

- Create user in DB first and send response
- AFTER your res.json() you perform the upload of the base64 image to cloudinary!
- If the upload succeeded: Replace the base64 Avatar of the user with the cloudinary URL!
- ` User.findByIdAndUpdate( newUser._id, { avatar_url: uploadResult.secure_url } ) ` 
- Do NOT send a response afterwards (you can just send ONE response to the user in a route)

This measure should reduce the response time on signup significantly. And you still get an image for your user back in the response which you can use in an `<img src="" />` tag!

Only risk: If the cloudinary upload fails, you are stuck with your base64 encoded string in the database. But that should be fine for just few entries. We just want to prevent storing ALL our images base64 encoded in the database, due the storage limit could get exceeded fast. A few base64 stored images should be fine. 

For the nerds: We could write a /reupload-avatars route which we call every now and then. That route will check all users in the database who still have a base64 encoded avatar and try to re-upload that one to cloudinary and replace then the base64 thing with the URL in our DB. This way we can clean and "compact" our DB entries regularly.

## Uploading two different file fields

Let's assume you wanna manage your user avatar + your family photo in your profile.

Now we will have two DIFFERENT file input fields we wanna upload. How do we do this?

##### Frontend Form

For the avatar we can set an input to select a single file.
For the family photo we set another input field.

```
<input type="file" name="avatar" accept="image/*" />
<input type="file" name="family" accept="image/*" />
```

##### Backend

Now you will have the base64 encoded image strings availabe in req.body as usual (req.body.avatar and req.body.family in this case).


### Uploading multiple files

Let's assume in a form you wanna upload images for a recipe of your favorite food.

#### Frontend

Simply add the "multiple" HTML attribute to your input field 

` <input type="file" name="recipe_images" accept="image/*" multiple />`

Now depending how you send the data, you need to prepare the data differently.

Here we assume, you manage the files in an array in state, e.g. `const [recipeImages, setRecipeImages] = useState([])`

If you put an onChange handler on your "multiple input" field, you will have now all files availabe in the event: `e.target.files`

Now you need to loop over that list of files and convert each file to a base64 string using the FileReader class (see code above).

Alternative: You do this conversion on submit of the whole form to make it a bit more efficient.

#### Backend

You will now receive the list of file strings in an array in `req.body.recipe_images`. 

Now you can loop over that array and upload each file to cloudinary, receiving an URL for each one. 

At the time of writing the author does not know of any batch uploading of an base64 array. But if you find something let me know :)


## Resources

- Cloudinary package: https://www.npmjs.com/package/cloudinary
- MongoDB ATLAS storage limits: https://www.mongodb.com/pricing
- MongoDB ATLAS - what will happen on reaching storage limit? https://docs.atlas.mongodb.com/reference/faq/storage/
