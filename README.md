# PHP Service Wrapper for MeTube

## Introduction

This is a (non-Dockerized) PHP app that "wraps around" [Alexta69](https://github.com/alexta69/)'s excellent [MeTube](https://github.com/alexta69/metube) Web GUI for [youtube-dl](https://github.com/ytdl-org/youtube-dl). For most use-cases, you can just directly use MeTube in a modern browser, and you don't need this project. This wrapper does add some features, depending on configuration, including:

* The ability to search YouTube for videos by name
* The ability to access youtube-dl retreived content over the web
* A clean-up script to remove videos (hopefully once they've been watched)

I wrote this wrapper for older devices that can't access YouTube, can't use youtube-dl natively, and can't render MeTube's web UI. Such devices would have to be capable of making relatively simple HTTP calls (POST and GET) and of playing back MP4 video containers with MP4 video and AAC audio tracks. Specifically, I wrote this for legacy Palm/HP [webOS](http://www.webosarchive.com/) devices. Other clients, such as older Macs, could conceivably use this service -- but I haven't tested. Although the specific container and media format required for webOS devices is prescribed in the code, both MeTube and youtube-dl are capable of helping you get other file formats; simply tweak `add.php` to suit your needs.

Note: this project was *not* created to steal content from YouTube, and the creator does not condone the use of this project for that purpose. My intent was to facilitate older devices streaming YouTube content and deliberate efforts were taken to ensure caches and temporary files are purged.

## Requirements

This PHP app was written for a Raspberry Pi, but scaled transparently to a mid-size Ubuntu VM in Azure. It probably works on a Mac, and its conceivable that you could run it on Windows as well, but you'd have to have PHP installed for IIS and I didn't build with that scenario in mind. In my environments I have installed:

* Apache2
* PHP 7.2
* PHP modules:
    + php-gd
    + php-xml
    + php7.2-curl

## Installation on Bare Metal

* Create a directory to store YouTube downloads, ensure the Apache user and group (usually www-data in Linux) has read and write access to that folder 
    + see this [Issue](https://github.com/alexta69/metube/issues/7) where I worked through permissions, so you don't repeat my mistakes!
* Get MeTube working with that download folder -- the Docker container actually works very well. Follow the [documentation](https://github.com/alexta69/metube/blob/master/README.md) on that project for guidance
* Configure an Apache Site for the PHP web app (or add a directory to an existing site)
* Clone this repo into that directory
* Make the clean-up script executable: `chmod +x youtube-cleanup.sh`
* Use `crontab -e` to establish the clean-up schedule.
    + Mine looks like: `*/15 * * * * /var/www/metube/youtube-cleanup.sh`
* Update the `config.php` file to point to:
    + Your MeTube endpoint URL -- which does not need to be exposed to the Internet (eg: localhost works fine)
    + Your MeTube port (eg: 8081)
    + The path to your downloads folder (established above)
    + If you want to use the YouTube Search feature, your Google API Key ([get your own for free here](https://developers.google.com/youtube/v3/getting-started))

That's it! Once the PHP app is running, you can begin to use it in even the simplest of web clients.

## Installation in Docker Container

* Community contributor Nomad84 has successfully dockerized the service wrapper and provided documentation on how he got it working. Those [instructions are provided here](Dockerize.md) as a reference only.

## webOS Usage

This service was made to work with the webOS app [MeTube](https://github.com/codepoet80/webos-metube) on long defunct Palm and HP devices, like Pre and Touchpad. You can use the hosted version, provided by [webOS Archive](http://www.webosarchive.com) without configuration in that client.

You can also use that client by hosting the service yourself. Follow the set-up directions above, then use the app's Preferences scene to change your Endpoint and API keys. As well as your own Google API key, this service includes a few shared secrets that need to be configured both in the app's Preferences, and in the `config.php` of your service instance.

See `config-sample.php` for details on setting up your shared secrets, to help keep your instance private.

(Note: that due to the age of webOS devices, this is not real security, only obfuscation. Since everything must be passed in the clear, it is vulnerable to common man-in-the-middle attacks. You probably aren't using your 10 year old Touchpad in public anyway, but just keep this in mind!)

## Alternate Client Usage

As discussed, the service was written with a specific client and use-case in mind. Its potentially flexible, but that would be up to you.

Additionally, the service was crafted in a way that attempts to protect ownership of the video. Again, the goal is not to take content from YouTube, but to allow older devices to use YouTube.

As an intended result of these two caveats, client requests may appear unnecessarily obscure. This is by design!

There are 4 main functions of this service, each will be discussed briefly, with some details of how to form a request. For examples of the code in Javascript, see the [metube-model.js](https://github.com/codepoet80/webos-metube/blob/main/app/models/metube-model.js) file in the [webOS client app](https://github.com/codepoet80/webos-metube) (webOS was a Javascript-based OS).

### Search

The search function works as a proxy for Google's YouTube search API, so that you don't have to embed your API key into older, and probably insecure, platforms. You send a search request to `search.php` with a GET call, and it sends you back the results from Google. If you set a `client_key` (or `debug_key`) value in your `config.php`, the client must send those values along with the request in the form of a header named `Client-Id`

The query string of the GET request should contain the query, and the number of desired results:
* `q=VIDEOTOSEARCHFOR`
* `maxResults=10`

Optionally, the client may send an alternate Google API Key, which will be honored by the service:
* `key=YOURGOOGLEAPIKEY`

The result will be the JSON payload from Google, as described in their [API documentation](https://developers.google.com/youtube/v3/guides/implementation/search), with Live videos filtered out.

### Add

Once a user has selected the video they want to see, they will want to send an **Add** call to MeTube to fetch and process the video on their behalf. MeTube uses youtube-dl to accomplish this, and this service  simply proxies MeTube. Client identification is the same as in search: if you set a `client_key` (or `debug_key`) value in your `config.php`, the client must send those values along with the request in the form of a header named `Client-Id`

Here we also add an additional layer of obfuscation -- both to protect the query content from being garbled in transmission, and to ensure the client is behaving as intended, where my intent is to not service unknown clients that might treat YouTube content unethically. With **Search**, we only asserted that the server knows identity of the client, using a client shared secret. With add, we will also try to validate that the client is trusted by the server with a server shared secret (with the caveat that no retro platform can have any assurance of trust!)

If the your `config.php` includes a value for `server_id`, this value should be hidden within the POST request. The entire payload of the POST request should be constructed as follows:

* The YouTube URL the user wants to watch, base64 encoded
* The `server_id` contiguously placed at random within the encoded string

If the request succeeds, a simple JSON response indicating "OK" will be returned. All other responses indicate an error.

### List

Once the video request has been added to MeTube, you will need to poll for the appearance of the processed file. MeTube does not issue a ticket, or maintain state for requests, but it does process requests in order. Assuming there aren't simultaneous (or close-to-simultaneous) client requests, your client can assume that the next file to appear in the list is the file for your most recent **Add** call.

Another caveat is that only new files are new -- if you send the same request again, you will not get a new file as a result. For that reason, maintaining the correct cleanup schedule (as set in cron, above) is important. Too aggressive, and you might delete a file you're using. Not aggressive enough, and you may deny service to a client. Mitigations were added in the webOS client to handle repeated requests within the clean-up schedule.

For these reasons, this service cannot scale. No attempt has been made to solve these problem, in order to limit use to a small number of users. In my case, the remaining webOS community is probably less than 30 people in the world!

A `list.php` request is a parameterless GET request. Client identification is the same as in search: if you set a `client_key` (or `debug_key`) value in your `config.php`, the client must send those values along with the request in the form of a header named `Client-Id`. The result will be a JSON structure that enumerates the .MP4 contents of the download folder you configured above.

### Play

Due to the constraints of my target client platform, the **Play** function has limited security and must be passed in-the-clear as a GET request. No headers can be included. As a result, the obfuscation is similar to the **Add** request:

If your `config.php` includes a value for `server_id`, this value should be hidden within the Query string. The entire query string to `play.php` should be constructed as follows:

* `video=FILENAME` -- this is the URL encoded version of the plain text filename of the video to play, as returned by the List function. 
* `requestid=` -- the encoded request with the `client_key` value and a | (pipe) prefixing the base64 encoded file name, as returned by the List function.

The video value, and the decoded filename in the request ID, must match. This deliberate obfuscation is to prevent a user from easily constructing a download query. It is up to the developer of the Client code to ensure the video is not retained on the client device.

## Debugging

Depending on how you deploy the various components, you may run into connectivity issues. While debugging it might be helpful to understand the call stack for each function:

* **Search**: client > php-service-wrapper > Google API
* **Add**: client > php-service-wrapper > MeTube > youtube-dl > YouTube.com
* **List**: client > php-service-wrapper
* **Play**: client > php-service-wrapper

If you're attempting to Dockerize this service wrapper, it must be able to communicate with the MeTube docker container over HTTP on the specified port.
