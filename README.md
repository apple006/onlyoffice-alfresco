# Alfresco ONLYOFFICE integration

This Share plugin enables users to edit Office documents within ONLYOFFICE from Alfresco Share. This will create a new **Edit in ONLYOFFICE** action within the document library for Office documents. This allows multiple users to collaborate in real time and to save back those changes to Alfresco.

Tested with Enterprise 5.0.\*, 5.1.\* and Community 5.1.\*

## Release Notes

### Version `1.3.0`

* OnlyOffice editor tab of web browser will be closed when timeout is reached, this is achieved by adding a new property in alfresco-global.properties:
```
  onlyoffice.timeout=120
```
  The value of this property stands for the desired timeout minutes, which means if there is no editing happened to the opened document after 120 minutes, the editor window will be closed.
* If property is not given, then no timeout minutes is set and editor will not be closed automatically

### Version `1.2.5`

* Documents that are above preview thresholds will no longer be previewed in Share
* Thresholds values can be set accordingly in alfresco-global.properties, the example settings are:
```
  onlyoffice.preview.document.size.threshold=10485760
  onlyoffice.preview.docx.threshold=8000
  onlyoffice.preview.doc.threshold=8000
  onlyoffice.preview.xlsx.threshold=10000
  onlyoffice.preview.xls.threshold=10000
  onlyoffice.preview.pptx.threshold=1000
  onlyoffice.preview.ppt.threshold=1000
```
* In the example settings above, document size threshold is 10Mb, the docx and doc threshold is max paragraphs number, the xlsx and xls threshold is max total rows of sheets, the pptx and ppt thresholds is max slides number.
* Any threshold's value will be 0 if they are missing in properties file. Therefore the corresponding threshold check won't be performed. For example, if `onlyoffice.preview.document.size.threshold` is not presen
t in alfresco-global.properties, then there won't be any limitation in terms of document size when user tries to preview a document.

### Version `1.1.0`


* Support for OnlyOffice version 4 (Tested with Alfresco Enterprise 5.1)
* Adds a few more Languages as per the ONLYOFFICE fork
* Limits the Edit in OnlyOffice button to certain mime types within Alfresco, rather than globally available as an action
* Adds a keep alive to make sure the share session does not expire when you have a window open
* Adds a new global property: `onlyoffice.lang`.  This will set the language of the editor.  E.g, for italian:
  ```
  onlyoffice.lang=it
  ```
* Wrapped some response writers in the try blocks

## Compiling

You will need:

* Java 7 SDK or above

* Gradle

* Parashift's alfresco amp plugin from here: https://bitbucket.org/parashift/alfresco-amp-plugin

* Run `gradle amp` from the `share` and `repo` directories


## Installation

### ONLYOFFICE

You will need an instance of ONLYOFFICE that is resolvable and connectable both from alfresco and any end clients. ONLYOFFICE must also be able to POST to alfresco directly.

The easiest way to start an instance of ONLYOFFICE is to use Docker: https://github.com/ONLYOFFICE/Docker-DocumentServer


### Alfresco

* Deploy the amp to both the repo and share end using alfresco-mmt or other methods

* Add the `onlyoffice.url` property to alfresco-global.properties:
  ```
  onlyoffice.url=http://documentserver/
  ```

*  OnlyOffice will make the connection to Alfresco on behalf of the client, so OnlyOffice needs to be able to talk to Alfresco.  In order for OnlyOffice to do this, Alfresco needs to generate what it thinks the external URL is.  Make sure that the following properties are set correctly in `alfresco-global.properties`

  ```
  alfresco.protocol=http
  alfresco.host=alfresco.yourcompany.local
  alfresco.port=8080
  alfresco.context=alfresco
  ```

* (Optional) set the default language that the editor will use.  By default the document editor should pick up the language of the browser, so it doesn't always need to be set.

  ```
  onlyoffice.lang=en
  ```

* (Optional) set another URL to use for PDF transformations.  By default when converting files to PDF it will use the `onlyoffice.url`:

  ```
  onlyoffice.transform.url=http://documentserver/
  ```

## How it works

The ONLYOFFICE integration follows the API documented here https://api.onlyoffice.com/editors/basic:

* User navigates to a document within Alfresco Share and selects the `Edit in ONLYOFFICE` action.
* Alfresco Share makes a request to the repo end (URL of the form: `/parashift/onlyoffice/prepare?nodeRef={nodeRef}`).
* Alfresco Repo end prepares a JSON object for Share with the following properties:
  * **docUrl**: the URL that ONLYOFFICE uses to download the document (includes the `alf_ticket` of the current user),
  * **callbackUrl**: the URL that ONLYOFFICE informs about status of the document editing,
  * **onlyofficeUrl**: the URL that the client needs to talk to ONLYOFFICE (given by the onlyoffice.url property),
  * **key**: the UUID+Modified Timestamp to instruct ONLYOFFICE whether to download the document again or not,
  * **docTitle**: the Title (name) of the document.
* Alfresco Share takes this object and constructs a page from a freemarker template, filling in all of those values so that the client browser can load up the editor.
* The client browser makes a request for the javascript library from ONLYOFFICE and sends ONLYOFFICE the docEditor configuration with the properties as above.
* ONLYOFFICE then downloads the document from alfresco and the user begins editing.
* ONLYOFFICE sends a POST request to the callback URL to inform Alfresco that a user is editing the document.
* Alfresco locks the document, but still allows other users with write access the ability to collaborate in real time with ONLYOFFICE by leaving the Action present.
* When all users and client browsers are finished, they close the editing window.
* After 10 seconds of inactivity, ONLYOFFICE sends a POST to the callback URL letting Alfresco know that the clients have finished.
* Alfresco downloads the new version of the document, replacing the old one.
