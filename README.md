# Deploy toxel

All commands are executed as operator.

* Open new browser window at https://USER_NAME.livemap24.com
* Login as ``USER_NAME``.
* Make sure you are logged in as ``USER_NAME`` (with operator access).

## Upload GTFS
* Drag and drop a GTFS zip-file anywhere in the browser window.
* When upload is finished, go to the home folder (case sensitive).
```cmd
/$ cd Home
```
* Execute the list files command
```cmd
/Home$ List-Files
```
* You should see your uploaded file in the list.

## Build toxel
* Execute the build toxel command
```cmd
/Home$ Build-Toxel File=FILE_NAME.zip
```
* List files with ``List-Files``
* Make sure FILE_NAME.toxel is created

## Deploy toxel
* Execute the deploy toxel command
```cmd
/Home$ Deploy-Toxel Files=FILE_NAME.toxel
```
* It is possible to deploy multiple files separated by comma.
