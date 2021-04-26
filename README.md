# File Uploading & Display - Fullstack

Find here summarized a mini guide how to incorporate file uplading from frontend to backend with a (free) file cloud provider.

We will use Cloudinary as the provider, which allows us (at the time of writing) 10GB+ storage for free.

We will use the example of an Avatar upload.

In this fullstack sample we will encode our images as base64 strings BEFORE uploading them to the API.

This way we can do not need to send "multipart formdata" which simplifies the workflow in the frontend a lot.

Also in the backend we do not need to parse binary data this way anymore, we can simply forward the received base64 to our file cloud provider.

## Steps 

* Signup to Cloudinary: https://cloudinary.com/users/register/free
* Free Plan & Pricing: https://cloudinary.com/pricing

### Backend part

* Install following packages: `npm i cloudinary dotenv` 
* Configure Cloudinary URL in a .env of your backend
  * The Cloudinary URL is similar to your MongoDB Atlas URL. A url to your own cloud including your user credentials
  * You find the Cloudinary Environment URL in the cloudinary dashboard after login

* Load the .env in your server.js file (if not done already):  `require("dotenv").config() `

* Adapt your model where you wanna attach an image URL
  * Example User Model: add a field "avatar_url" (String)

* Upload route
  * Extract / split the avatar file and normal JSON data from req.body: `const { avatar, ...userData } = req.body `
  * Upload the avatar string to cloudinary
    * `const result = await cloudinary.upload.upload( avatar )`
  * Store the received URL in your model
    * e.g. `const userNew = await User({ ...userData, avatar_url: result.secure_url }) `
    * Cloudinary will provide you with TWO urls in its response: url and secure_url
     * url will be reachable by http:// and secure_url will be reachable by https://. So it is advisable to always use the secure one 
  * Return the created user to the frontend using res.json()

* Test File upload against your route from Insomnia
  * Use JSON as Body format
  * Provide normal JSON data
  * Convert some avatar file to a base64 dataUri, e.g. here: https://www.base64-image.de/
  * Paste the dataUri into your Body
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
  * e.target.files contains the binary files. These we now need to convert to DataURI Strings using the builtin FileReader class
  * ```
    let fileSelected = e.target.files[0]  // grab selected file

    if(!fileSelected) return

    let fileReader = new FileReader()
    fileReader.readAsDataURL( fileSelected ) // concert to base64 encoded string
    // wait until file is fully loaded / converted to base64
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
  * The simplest way is to simply not handle the file input by React-Hook-Form

* Submitting
  * If we wanna send mixed data, we cannot simply pass in JSON data anymore to Axios
  * We need to pass in a FormData object
  * Example:
    ```
      onSubmit = (jsonData) => {
      
        jsonData.avatar = avatarPreview // merge the avatar string into our data

        // signup user in backend
        try {
          let response = await axios.post('http://localhost:5000/users', jsonData)
          console.log("Response: ", response.data) // => signed up user
          history.push('/users')  
        }
        // handle error
        catch(errAxios) {
          console.log(errAxios.response && errAxios.response.data)
        }
      }
    ```

### Uploading multiple files

Let's assume in a form you wanna upload images for a recipe of your favorite food.

#### Frontend

Simply add the "multiple" HTML attribute to your input field 

` <input type="file" name="recipe_images" accept="image/*" multiple />`

#### Backend

Will follow...


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

Will follow...


## Resources

- Cloudinary package: https://www.npmjs.com/package/cloudinary
- MongoDB ATLAS storage limits: https://www.mongodb.com/pricing
- MongoDB ATLAS - what will happen on reaching storage limit? https://docs.atlas.mongodb.com/reference/faq/storage/
