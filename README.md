# FileProvider

>This Swift library provide a swifty way to deal with local and remote files and directories in a unified way.

[![Swift Version][swift-image]][swift-url]
[![License][license-image]][license-url]
[![Platform](https://img.shields.io/badge/Platform-iOS%2C%20macOS%2C%20tvOS-lightgray.svg)]()

[![Build Status][travis-image]][travis-url]
[![CocoaPods Compatible](https://img.shields.io/cocoapods/v/FileProvider.svg)](https://cocoapods.org/pods/FileProvider)
[![codebeat badge][codebeat-image]][codebeat-url]

<!--- 

[![codecov](https://codecov.io/gh/amosavian/FileProvider/branch/master/graph/badge.svg)](https://codecov.io/gh/amosavian/FileProvider)
[![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage) 
---> 

This library provides implementaion of WebDav and SMB2 (incomplete) and local files.

All functions are async calls and it wont block your main thread.

Local and WebDAV providers are fully tested and can be used in production environment.

## Features

- [x] **LocalFileProvider** a wrapper around `FileManager` with some additions like searching and reading a portion of file.
- [x] **WebDAVFileProvider** WebDAV protocol is defacto file transmission standard, replaced FTP.
- [x] **DropboxFileProvider** A wrapper around Dropbox Web API. For now it has limitation in uploading files up to 150MB.
- [ ] **SMBFileProvider** SMB2/3 introduced in 2006, which is a file and printer sharing protocol originated from Microsoft Windows and now is replacing AFP protocol on MacOS. I implemented data types and some basic functions but *main interface is not implemented yet!* SMB1/CIFS is depericated and very tricky to be implemented
- [ ] **FTPFileProvider** while deprecated in 1990s, it's still in use on some Web hosts.
- [ ] **AmazonS3FileProvider** 

## Requirements

- **Swift 3**
- iOS 8.0 , OSX 10.10
- XCode 8.0

Legacy version is available in swift-2 branch

## Installation

### Cocoapods / Carthage / Swift Package Manager

FileProvider supports both CocoaPods. 

Add this line to your pods file:

	pod "FileProvider"

### Git
To have latest updates with ease, use this command on terminal to get a clone:

	git clone https://github.com/amosavian/FileProvider FileProvider
	
You can update your library using this command in FileProvider folder:

	git pull

if you have a git based project, use this command in your projects directory to add this project as a submodule to your project:

	git submodule add https://github.com/amosavian/FileProvider FileProvider

### Manually
Copy Source folder to your project and Voila!

## Usage

Each provider has a specific class which conforms to FileProvider protocol and share same syntax

### Initialization

For LocalFileProvider if you want to deal with `Documents` folder

	let documentsProvider = LocalFileProvider()

is equal to:
	    
	let documentPath = NSSearchPathForDirectoriesInDomains(NSSearchPathDirectory.DocumentDirectory, NSSearchPathDomainMask.UserDomainMask, true);
	let documentsURL = URL(fileURLWithPath: documentPath);
	let documentsProvider = LocalFileProvider(baseURL: documentsURL)

You can't change the base url later. and all paths are related to this base url by default.

For remote file providers authentication may be necessary:

	let credential = URLCredential(user: "user", password: "pass", persistence: .permanent)
	let webdavProvider = WebDAVFileProvider(baseURL: URL(string: "http://www.example.com/dav")!, credential: credential)

* In case you want to connect non-secure servers for WebDAV (http) in iOS 9+ / macOS 10.11+ you should disable App Transport Security (ATS) according to [this guide.](https://gist.github.com/mlynch/284699d676fe9ed0abfa)

* For Dropbox, user is clientID and password is Token which both must be retrieved via [OAuth2 API of Dropbox](https://www.dropbox.com/developers/reference/oauth-guide). There are libraries like [p2/OAuth2](https://github.com/p2/OAuth2) or [OAuthSwift](https://github.com/OAuthSwift/OAuthSwift) which can facilate the procedure to retrieve token.
	
For interaction with UI, set delegate variable of `FileProvider` object

You can use `absoluteURL()` method if provider to get direct access url (local or remote files) for some file systems which allows to do so (Dropbox doesn't support and returns path simply wrapped in URL)

### Delegates

For updating User interface please consider using delegate method instead of completion handlers. Delegate methods are guaranteed to run in main thread to avoid bugs.

It's simply three method which indicated whether the operation failed, succeed and how much of operation has been done (suitable for uploading and downloading operations).

Your class should conforms `FileProviderDelegate` class:

	override func viewDidLoad() {
		documentsProvider.delegate = self as FileProviderDelegate
	}
	
	func fileproviderSucceed(_ fileProvider: FileProviderOperations, operation: FileOperation) {
		switch operation {
		case .copy(source: let source, destination: let dest):
			print("\(source) copied to \(dest).")
		case .remove(path: let path):
			print("\(path) has been deleted.")
		default:
			break
		}
	}
	
    func fileproviderFailed(_ fileProvider: FileProviderOperations, operation: FileOperation) {
    	switch operation {
		case .copy(source: let source, destination: let dest):
			print("copy of \(source) failed.")
		case .remove(path: let path):
			print("\(path) can't be deleted.")
		default:
			break
		}
    }
	
    func fileproviderProgress(_ fileProvider: FileProviderOperations, operation: FileOperation, progress: Float) {
		switch operation {
		case .copy(source: let source, destination: let dest):
			print("Copy\(source) to \(dest): \(progress * 100) completed.")
		default:
			break
		}
	}

**Note:** `fileproviderProgress()` delegate method is not called by `LocalFileProvider` currently. 

It's recommended to use completion handlers for error handling or result processing.

#### Controlling file operations

You can also implement `FileOperationDelegate` protocol to control behaviour of file operation (copy, move/rename, remove and linking), and decide which files should be removed for example and which won't. 

`fileProvider(shouldDoOperation:)` method is called before doing a operation. You sould return `true` if you want to do operation or `false` if you want to stop that operation.

`fileProvider(shouldProceedAfterError:, operation:)` will be called if an error occured during file operations. Return `true` if you want to continue operation on next files or `false` if you want stop operation further. Default value is false if you don't implement delegate.

**Note: these methods will be called for files in a directory and its subfolders recursively.**

### Directory contents and file attributes

There is a `FileObject` class which holds file attributes like size and creation date. You can retrieve information of files inside a directory or get information of a file directly.

For a single file:

	documentsProvider.attributesOfItem(path: "/file.txt", completionHandler: {
	    (attributes: LocalFileObject?, error: ErrorType?) -> Void in
		if let attributes = attributes {
			print("File Size: \(attributes.size)")
			print("Creation Date: \(attributes.createdDate)")
			print("Modification Date: \(attributes.modifiedDate)")
			print("Is Read Only: \(attributes.isReadOnly)")
		}
	})

To get list of files in a directory:

	documentsProvider.contentsOfDirectory(path: "/", 	completionHandler: {
	    (contents: [LocalFileObject], error: ErrorType?) -> Void in
		for file in contents {
			print("Name: \(attributes.name)")
			print("Size: \(attributes.size)")
			print("Creation Date: \(attributes.createdDate)")
			print("Modification Date: \(attributes.modifiedDate)")
		}
	})

To get size of strage and used/free space:

	func storageProperties(completionHandler: {(total: Int64, used: Int64) -> Void in
	    print("Total Storage Space: \(total)")
	    print("Used Space: \(used)")
	    print("Free Space: \(total - used)")
	})
	
* if this function is unavailable on provider or an error has been occurred, total space will be reported `-1` and used space `0`

### Change current directory

	documentsProvider.currentPath = "/New Folder"
	// now path is ~/Documents/New Folder
	
You can then pass "" (empty string) to `contentsOfDirectory` method to list files in current directory.

### Creating File and Folders

Creating new directory:

	documentsProvider.create(folder: "new folder", at: "/", completionHandler: nil)

Creating new file from data stream:

	let data = "hello world!".data(encoding: String.encoding.utf8)
	let file = FileObject(name: "old.txt", createdDate: Date(), modifiedDate: Date(), isHidden: false, isReadOnly: true)
	documentsProvider.create(file: file, at: "/", contents: data, completionHandler: nil)

### Copy and Move/Rename Files

Copy file old.txt to new.txt in current path:

	documentsProvider.copyItem(path: "new folder/old.txt", to: "new.txt", overwrite: false, completionHandler: nil)

Move file old.txt to new.txt in current path:

	documentsProvider.moveItem(path: "new folder/old.txt", to: "new.txt", overwrite: false, completionHandler: nil)

**Note:** To have a consistent behaviour, create intermediate directories first if necessary.

### Delete Files

	documentsProvider.removeItem(path: "new.txt", completionHandler: nil)

***Caution:*** This method will delete directories with all it's content recursively.

### Retrieve Content of File

There is two method for this purpose, one of them loads entire file into NSData and another can load a portion of file.

	documentsProvider.contents(path: "old.txt", completionHandler: {
		(contents: Data?, error: ErrorType?) -> Void
		if let contents = contents {
			print(String(data: contents, encoding: String.encoding.utf8)) // "hello world!"
		}
	})
	
If you want to retrieve a portion of file you can use `contents` method with offset and length arguments. Please note first byte of file has offset: 0.

	documentsProvider.contents(path: "old.txt", offset: 2, length: 5, completionHandler: {
		(contents: Data?, error: ErrorType?) -> Void
		if let contents = contents {
			print(String(data: contents, encoding: String.encoding.utf8)) // "llo w"
		}
	})

### Write Data To Files

	let data = "What's up Newyork!".data(encoding: String.encoding.utf8)
	documentsProvider.writeContents(path: "old.txt", content: data, atomically: true, completionHandler: nil)

### Operation Handle

Creating/Copying/Deleting functions return a `OperationHandle` for remote operations. It provides progress and a `.cancel()` method which allows you to cancel operation in midst.

It's not supported by native `(NS)FileManager` so `LocalFileProvider`, but this functionality will be added to future `PosixFileProvider` class.

### Monitoring FIle Changes

You can monitor updates in some file system (Local and SMB2), there is three methods in supporting provider you can use to register a handler, to unregister and to check whether it's being monitored or not. It's useful to find out when new files added or removed from directory and update user interface. The handler will be dispatched to main threads to avoid UI bugs with a 0.25 sec delay.

	documentsProvider.registerNotifcation(path: provider.currentPath)
	{
		// calling functions to update UI 
	}
	
	// To discontinue monitoring folders:
	documentsProvider.unregisterNotifcation(path: provider.currentPath)

* **Please note** in LocalFileProvider it will also monitor changes in subfolders. This behaviour can varies according to file system specification.

## Contribute

We would love for you to contribute to **FileProvider**, check the `LICENSE` file for more info.

## Projects in use

* [EDM - Browse and Receive Files](https://itunes.apple.com/us/app/edm-browse-and-receive-files/id948397575?ls=1&mt=8)
* [File Manager - PDF Reader & Music Player](https://itunes.apple.com/us/app/file-manager-pdf-reader-music/id1017809685?ls=1&mt=8)

If you used this library in your project, you can open an issue to inform us.

## Meta

Amir-Abbas Mousavian  – [@amosavian](https://twitter.com/amosavian)

Distributed under the MIT license. See `LICENSE` for more information.

[https://github.com/amosavian/](https://github.com/amosavian/)

[swift-image]:https://img.shields.io/badge/swift-3.0-orange.svg
[swift-url]: https://swift.org/
[license-image]: https://img.shields.io/badge/License-MIT-blue.svg
[license-url]: LICENSE
[codebeat-image]: https://codebeat.co/badges/7b359f48-78eb-4647-ab22-56262a827517
[codebeat-url]: https://codebeat.co/projects/github-com-amosavian-fileprovider
[travis-image]: https://img.shields.io/travis/amosavian/FileProvider/master.svg
[travis-url]: https://travis-ci.org/amosavian/FileProvider