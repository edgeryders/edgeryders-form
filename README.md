# Edgeryders Form


## 1. Overview

**Description.** A standalone, Vue.js based survey form builder software that posts its collected data as topics to a [Discourse](https://discourse.org/) forum. It is meant as a low-barrier way for potential new users to interact with a Discourse forum. Their first post is just about filling out a simple, minimalist looking form, and afterwards they already have a full Discourse account, will receive e-mail notifications about responses to their topic, and can engage further if they feel like it.

**Requirements.** Out of the box, the software works only with the modified Discourse installations of [edgeryders.eu](https://edgeryders.eu/) and its associated forums called "Edgeryders Communities". You can set up an equivalent installation with a separate SSO provider and SSO client Discourse installation, as per the instructions below. But if you operate a normal Discourse forum with direct login, it will be simpler to adapt this software accordingly.

**Architecture.** The application is a single-page web application made with Vue.js, using the API of two Discourse installations as its backend. One Discourse installation is the SSO provider, needed for creating a new user account. The other Discourse installation is the actual forum, needed to post a survey response under the Discourse account created for the user.


## 2. Installation

Installation is not trivial as this software needs a few modifications on the Discourse side to function (two API extension plugins plus configuration).

**1. Make sure all your Discourse sites use HTTPS**

Since we transfer API keys and passwords to them via API, this is strongly encouraged. Also if your Discourse Form Vue.js application is served by HTTPS, the Discourse APIs it accesses also need to use HTTPS or the browser will refuse to send the API requests due to "mixed content error".

**2. Install the multisite account Discourse plugin**

This plugin has to be installed on the Discourse instance used as the SSO provider. For the plugin source code and installation instructions, [see here](https://github.com/edgeryders/discourse/tree/multisite/plugins/edgeryders-multisite-accounts).

Note that this plugin needs a configuration section in `config/secrets.yml` of your Discourse installation where you define the `auth_key` (an anti-spam measure, [see here](https://edgeryders.eu/t/-/7904#heading--3-4)) and also enter the master API keys (called the "All Users" API keys in the Discourse backend) of all SSO client Discourse instances on which you want your installation(s) of this survey software to post. The configuration section will look like this:

```
production:
  auth_key: …
  sso_provider:
    hostname: login.example.com
    api_key: …
  communities:
    - hostname: forum1.example.com
      api_key: …
    - hostname: forum2.example.com
      api_key: …
    - hostname: forum3.example.com
      api_key: …
```


**3. Install the permit-api-cors Discourse plugin**

This plugin has to be installed on the Discourse instance providing the actual forum. For the plugin source code and installation instructions, [see here](https://github.com/edgeryders/permit-api-cors).

**4. Enable and configure CORS in Discourse**

The following steps have to be done in both Discourse instances (SSO provider and SSO client).

1. **Enable CORS.** In a standalone ("no Docker") Discourse installation, edit the file `config/discourse.conf` and set `enable_cors = true` in there. Restart your Discourse web server afterwards. In a Docker-based installation, the configuration goes into `app.yml` in the project root folder in a slightly different form; we did not test that yet.

2. **Add the CORS origin.** Add the domain that will host this Edgeryders Form Vue.js application to the `cors_origins` setting via the Discourse admin panel.

3. **Adapt additional CORS configurations.** Non-Discourse software may also require a CORS configuration, for example to access statically served files under `https://example.com/.well-known/`. If so, make sure that your separate CORS configuration at webserver level does not conflict with that of Discourse by appending header values (which [browsers don't accept](http://www.w3.org/TR/cors/#access-control-allow-origin-response-header)) or overwriting header values. For Apache, see the tips [here](https://stackoverflow.com/a/47383394) and [here](https://stackoverflow.com/a/23998863) to get this to work.

**5. Install the Edgeryders Form software**

1. Clone the repository into a folder for serving a website:

    ```
    git clone https://github.com/edgeryders/edgeryders-form.git
    ```
2. Create a file `.env.production` in the root directory of the Git repository, with the following content adapted to the settings you need:

    ```
    # Our custom user creation endpoint at the Discourse SSO provider site.
    VUE_APP_DISCOURSE_USER_URL=http://login.example.com/multisite_account.json
    # auth_key, as configured in our multisite accounts Discourse plugin.
    VUE_APP_DISCOURSE_AUTH_KEY=<auth_key>
    # Domain of the Discourse forum to post survey responses on.
    VUE_APP_DISCOURSE_DOMAIN=forum1.example.com
    # API endpoint for posting survey responses.
    VUE_APP_DISCOURSE_TOPIC_URL=https://forum1.example.com/posts.json
    # Numeric category ID of a category where to post the survey responses.
    # The category must allow all users to post. To find the ID, see https://edgeryders.eu/t/-/6647#heading--2-1
    VUE_APP_DISCOURSE_CATEGORY=343
    ```

3. Install required package dependencies:

    ```
    yarn install
    ```

4. Compile and minify the site for production use:

    ```
    yarn run build
    ```

5. Configure your webserver to serve the website as static files, using the `dist/` sub-directory as document root.

6. Enable HTTPS for serving the website. Not strictly required though.


## 3. Building the survey form

The form slide types, texts and buttons, and structure can be configured with JSON files.

A simple example is below:

```
{
  "brand": {                            // controls the values presented on the landing page
    "title": "Some title",              // the title of the page
    "logoSrc": "/favicon.ico",          // path to the desired favicon
    "imageSrc": "/logo.png",            // path to the brand image in the bottom left of the page
    "beginText": "Begin",               // text for the button to enter the form
    "beginTitle": "Let's begin!"        // title text for the button to enter the form
  },

  slides": [{                           // an array of slides to present in order
    "title": "A title",                 // the title of the slide
    "subtitle": "A subtitle",           // a subtitle
    "body": "Some copy",                // a slide description
    "diagram": {                        // an image to include beneath the description
      "src": "https://map.org/map.png", // the src of the image
      "alt": "Some image"               // the alt of the image
    },
    "index": 1,                         // an index to include in the submitted form. Forms with fields must include this value to be included in the final response.
    "backText": "Back",                 // the text of the back button. Set to null to disallow going back from this slide
    "backTitle": "Go back",             // the title text of the back button
    "nextText": "Next",                 // the text of the next button. Set to null to disallow going forward from this slide
    "nextTitle": "Go forward",          // the title text of the next button
    "cancelTitle": "To the beginning"   // the title text of the cancel button
    "settings": {
      "omitFields": true
    },
    "fields": [{                        // a list of fields to include in a form on the slide. leave as '[]' to make an informational slide.
      "name": "name",                   // the name attribute of the field input
      "type": "text",                   // the type attribute of the field input
      "placeholder": "Name/nickname",   // the placeholder of the field input
      "half": true,                     // set to true to set this and the next field on the same line
      "required": true,                 // display the given error if field is not filled out on next
      "autofocus": true,                // select this field input automatically when navigating to this slide
      "error": "This is required"       // error to display if required field is not filled in
    }],

    "slideDefaults": {                  // all slides will fall back to the defaults given in this object if they are not defined on the slide themself.
      ...
    },

    "settings" [{                       // various settings used to define alternate behaviour for this slide
      "invert": true                    // display this slide with primary color background and secondary color foreground
      "omitFields": true                // don't include the field names with this response
    }]

    "messages" {                        // error messages to display when something goes wrong
      "networkError": "oops!"           // message when unable to get a response from the remote server
      "default": "something went wrong" // default error message
    }
  }]
}
```

## 4. Development

* To compile and hot-reload the site during development:

   ```
    yarn run serve
    ```

* To run the tests:

    ```
    yarn run test
    ```

* To lint and fix files:

    ```
    yarn run lint
    ```

* To customize your configuration, see [Configuration Reference](https://cli.vuejs.org/config/).


## 5. Documentation

API documentation:

* **multisite_account.json.** The custom Discourse API extension created by Edgeryders that allows to create a new account with Discourse as a SSO provider. See: [API documentation](https://edgeryders.eu/t/-/7904#heading--3-4), [plugin source code](https://github.com/edgeryders/discourse/tree/multisite/plugins/edgeryders-multisite-accounts).

* **posts.json.** Part of the standard [Discourse API](https://docs.discourse.org/), and used to create a new topic with the survey response, under a Discourse account created for that user. See: [API documentation](https://docs.discourse.org/#tag/Posts/paths/~1posts.json/post).
