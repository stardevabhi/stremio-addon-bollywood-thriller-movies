# Addon for Stremio to retrieve select bollywood thriller movies from youtube.

## steps to re-create the addon on windows

#### step 1: Install nodejs and npm

First make sure you have node.js and npm installed. Node.js installer can be downloaded from https://nodejs.org. Run the installer and nodejs and npm will be installed. After installation restart the computer. To check if nodejs and npm are installed properly, open the command line interface by pressing windows key+R and entering cmd in it.

Now to check the nodejs installation run the below command:

```javascript
node -v
```

this should print the nodejs version number like v8.11.1


To check the npm installation, enter the below command

```javascript
npm -v
```

This should print the version number like 5.6.0


#### step 2: Install dependency modules

Now, create a directory/folder in which you want the addon to be created and switch to it in command-line interface.

Run the following command:

```javascript
npm init -yes
````

It will create a package.json file in that directory.

Now, install stremio-addons module as dependency.

```javascript
npm install stremio-addons --save
```

Install puppeteer module dependency.

```javascript
npm install puppeteer --save
```


#### step 3: Re-create the addon code

Create a file named index.js and enter in, the code below:

```javascript
const puppeteer = require('puppeteer');
const Stremio = require('stremio-addons');
 

let result;
let href_result;
let imdb_result;
let item_array = [];
let href_item_array = [];
let imdb_item_array = [];
let imdb_ids = [];

const filters = ['New Hindi Movies', 'Bollywood Full Movies', 'Latest Hindi Full Movie', 'full movie in hd', 'Deleted video', 'Private video', 'Full Movies', 'Full Movie', 'Latest bollywood Movies', 'With Eng Subtitles', 'with Eng Subs', 'English Subtitles', 'Eng Subs', 'Hindi Movies', 'hindi movie', 'Superhit', 'ft.', 'Movies' , 'Hindi DvdRip', 'Movie', 'HD', 'bollywood', 'blockbuster', ' -', '- ', ' - '];


```

The code above include the required modules in our file, declare some variables and set the filters.


The code below define the scrape function, which scrapes the youtube playlist and search the  youtube video titles in google to retrieve the corresponding imdb ID. Append the below code to index.js file.

```javascript

let scrape = async () => {


	 

	const browser = await puppeteer.launch({headless: true,
	//slowMo: 2000
	 });  
	const page = await browser.newPage();

	const playlists = ['PLSVSKj5sfgxNqH8nybYOtidvcwX1cOmt-' ]

	for(let i=0; i<playlists.length;i++) {

	await page.goto('https://www.youtube.com/playlist?list=' + playlists[i], {waitUntil: 'load'});
	await page.waitFor(1000);

 	//scrape
 	console.log('Preparing data to load, please wait...' + "\n");
	 
	result = await page.evaluate(() => {

			
			let title_array = document.querySelectorAll('ytd-section-list-renderer#primary #contents .ytd-playlist-video-list-renderer #video-title');
			let titles = [];

			title_array.forEach(function(item, index) {
				titles.push(item.innerText);
			});
			
			return titles

		}); //evaluate

	 href_result = await page.evaluate(() => {

			//window.alert('s');
			let href_array = document.querySelectorAll('ytd-section-list-renderer#primary #contents .ytd-playlist-video-list-renderer #content > a');
			let hrefs = [];
			
			href_array.forEach(function(item, index) {
					const href_value =  item.getAttribute('href');
					const splitted_href_value = href_value.split('v=');
					const splitted_2_href_value_array = splitted_href_value[1].split('&');
					const video_id = splitted_2_href_value_array[0];
					hrefs.push(video_id);
			});
			
			return hrefs

		}); //evaluate


	//now search google for imdb titles
	
	for(let i = 0; i < result.length; i++ ) {

		 
	    await page.goto('https://google.com/' , {waitUntil: 'load'});
		 
		//hardcode these patterns
		result[i] = result[i].replace(/\(|\)/g, '');
		result[i] = result[i].replace(/\[|\]/g, '');
		result[i] = result[i].replace(/\|/g, '');
		result[i] = result[i].replace(/\{|\}/g, '');
		
		


		 filters.forEach((item, index)=> {

		 	var regex = item;
		 	var re = new RegExp(regex, "gi");

		 	if( result[i].toLowerCase().indexOf( item.toLowerCase() ) != -1 ) {
		 		//if filter word exists in title
		 		result[i] = result[i].replace(re, '');

		 	}

		 });//forEach

		 if( result[i].trim().length > 0) {

		 result[i] = result[i].replace(/ +/g, ' ');
		 //console.log(result[i]);

		 await page.type('input[name="q"]',  'imdb ' + result[i] );
		 await page.waitFor(500);
		 await page.keyboard.press('Enter');
		 await page.waitForSelector('.g h3.r a');

		 imdb_result = await page.evaluate(() => {

	 	 const imdb_title = document.querySelectorAll('#rso > div > div > div > div > div > h3 > a');
		 let imdb_id;
		 	 


		 		for(let i=0; i< imdb_title.length; i++) {
		 			const item1 = imdb_title[i];
		 		if(imdb_title[i].toString().indexOf('imdb.com/title/') != -1) {
		 			const imdb_title_splitted = item1.toString().split('/');
		 			imdb_id = imdb_title_splitted[4];
		 			break;
		 		}



		 	}; //foreach

		 	return imdb_id;

		 });//evaluate

		} else {
			imdb_result = '';

		}


		 imdb_ids.push(imdb_result);

	 } //for

	item_array = item_array.concat(result);
	href_item_array = href_item_array.concat(href_result);
	imdb_item_array = imdb_item_array.concat(imdb_ids);
 
	 } //for


	await browser.close();


	return  { item_array, href_item_array, imdb_item_array };

}; //scrape

scrape().then((value) => {

	let json_str = '{';

	value.item_array.forEach(function(item, index) {

	if( value.item_array[index] != "" ) {
		//create json
		
		
		json_str+= '"' + value.imdb_item_array[index] + '" : { "yt_id" : "' + value.href_item_array[index] + '" },';

	}//if

	});//value.item_array


json_str = json_str.substr(0, (json_str.length -1) );

json_str+="}";


stremioFn(json_str);

});//scrape.then


```

In the code above you also noticed the 'stremioFn' function, it is called with the json dataset as a parameter. This function is defined below, append this to index.js file.


```javascript
let stremioFn = async (json_str) => {

process.env.STREMIO_LOGGING = true;  


var manifest = { 
    "id": "org.stremio.bollywoodthrillers",
    "version": "1.0.0",

    "name": "Bollywood Thriller Movies",
    "description": "Addon for Bollywood Thriller Movies",
    "icon": "https://raw.githubusercontent.com/stardevabhi/stremio-addon-bollywood-thriller-movies/master/icons/bollywood-thriller.png", 
    "logo": "https://raw.githubusercontent.com/stardevabhi/stremio-addon-bollywood-thriller-movies/master/icons/bollywood-thriller.png", 
    "background": "https://raw.githubusercontent.com/stardevabhi/stremio-addon-bollywood-thriller-movies/master/icons/bollywood-thriller-1920x1080.png",

    // Properties that determine when Stremio picks this add-on
    "types": ["movie"], // your add-on will be preferred for those content types
    "idProperty": "imdb_id", // the property to use as an ID for your add-on; your add-on will be preferred for items with that property; can be an array
    // We need this for pre-4.0 Stremio, it's the obsolete equivalent of types/idProperty
    "filter": { "query.imdb_id": { "$exists": true }, "query.type": { "$in":["movie"] } }
};

var js_string = json_str;

var dataset =  JSON.parse(js_string);
console.log('Data loaded successfully!' + "\n");
 
var methods = { };
var addon = new Stremio.Server(methods, manifest);

if (module.parent) {
    module.exports = addon
} else {
    var server = require("http").createServer(function (req, res) {
        addon.middleware(req, res, function() { res.end() }); // wire the middleware - also compatible with connect / express
    }).on("listening", function()
    {
        console.log("Bollywood Thriller Movies Addon listening on "+server.address().port);
    }).listen(process.env.PORT || 7001);
}


// Streaming
methods["stream.find"] = function(args, callback) {
    if (! args.query) return callback();

    //callback(null, [dataset[args.query.imdb_id]]); // Works only for movies
    
    var key = [args.query.imdb_id, args.query.season, args.query.episode].filter(function(x) { return x }).join(" ");
    callback(null, [dataset[key]]);
};

// Add sorts to manifest, which will add our own tab in sorts
manifest.sorts = [{prop: "popularities.bollywoodthrillers", name: "Bollywood Thrillers",types:["movie"]}];

// To provide meta for our movies, we'll just proxy the official cinemeta add-on
var client = new Stremio.Client();
client.add("http://cinemeta.strem.io/stremioget/stremio/v1");


methods["meta.find"] = function(args, callback) {
    var ourImdbIds = Object.keys(dataset).map(function(x) { return x.split(" ")[0] });
    args.query = args.query || { };
    args.query.imdb_id = args.query.imdb_id || { $in: ourImdbIds };
    client.meta.find(args, function(err, res) {
    if (err) console.error(err);
        callback(err, res ? res.map(function(r) { 
            r.popularities = { bollywoodthrillers: 10000 }; // we sort by popularities.bollywoodthrillers, so we should have a value
            return r;
        }) : null);
    });
}

}
```


The index.js file is complete now. Let's take a look at the code blocks added to 'stremioFn' function.

The 'stremioFn' specify the manifest: 


```javascript


var manifest = { 
    "id": "org.stremio.bollywoodthrillers",
    "version": "1.0.0",

    "name": "Bollywood Thriller Movies",
    "description": "Addon for Bollywood Thriller Movies",
    "icon": "https://raw.githubusercontent.com/stardevabhi/stremio-addon-bollywood-thriller-movies/master/icons/bollywood-thriller.png", 
    "logo": "https://raw.githubusercontent.com/stardevabhi/stremio-addon-bollywood-thriller-movies/master/icons/bollywood-thriller.png", 
    "background": "https://raw.githubusercontent.com/stardevabhi/stremio-addon-bollywood-thriller-movies/master/icons/bollywood-thriller-1920x1080.png",

    // Properties that determine when Stremio picks this add-on
    "types": ["movie"], // your add-on will be preferred for those content types
    "idProperty": "imdb_id", // the property to use as an ID for your add-on; your add-on will be preferred for items with that property; can be an array
    // We need this for pre-4.0 Stremio, it's the obsolete equivalent of types/idProperty
    "filter": { "query.imdb_id": { "$exists": true }, "query.type": { "$in":["movie"] } }
};

```

parse the scraped dataset


```javascript
var js_string = json_str;

var dataset =  JSON.parse(js_string);
console.log('Data loaded successfully!' + "\n");
```
 

Create the server which listens on port 7001, with empty methods which are defined later: 

```javascript

var methods = { };
var addon = new Stremio.Server(methods, manifest);

if (module.parent) {
    module.exports = addon
} else {
    var server = require("http").createServer(function (req, res) {
        addon.middleware(req, res, function() { res.end() }); // wire the middleware - also compatible with connect / express
    }).on("listening", function()
    {
        console.log("Bollywood Thriller Movies Addon listening on "+server.address().port);
    }).listen(process.env.PORT || 7001);

```


stremio 'stream.find' method is defined here:


```javascript
// Streaming
methods["stream.find"] = function(args, callback) {
    if (! args.query) return callback();

    //callback(null, [dataset[args.query.imdb_id]]); // Works only for movies
    
    var key = [args.query.imdb_id, args.query.season, args.query.episode].filter(function(x) { return x }).join(" ");
    callback(null, [dataset[key]]);
};
```


Code below adds the tab named on our addon in the sorts section

```javascript

// Add sorts to manifest, which will add our own tab in sorts
manifest.sorts = [{prop: "popularities.bollywoodthrillers", name: "Bollywood Thrillers",types:["movie"]}];

```

Code below uses the official cinemeta addons to provide meta for our videos and define the 'meta.find' method.

```javascript

// To provide meta for our movies, we'll just proxy the official cinemeta add-on
var client = new Stremio.Client();
client.add("http://cinemeta.strem.io/stremioget/stremio/v1");


methods["meta.find"] = function(args, callback) {
    var ourImdbIds = Object.keys(dataset).map(function(x) { return x.split(" ")[0] });
    args.query = args.query || { };
    args.query.imdb_id = args.query.imdb_id || { $in: ourImdbIds };
    client.meta.find(args, function(err, res) {
    if (err) console.error(err);
        callback(err, res ? res.map(function(r) { 
            r.popularities = { bollywoodthrillers: 10000 }; // we sort by popularities.bollywoodthrillers, so we should have a value
            return r;
        }) : null);
    });
}

}

```

 All done!

 #### step 4: Testing!

 Now, when all code is in place, it's time to see it working. Save the file and run it using the command below.

 
```javascript
 node index.js
```

Open the stremio app and navigate to addons section via settings > addons and in the 'Addon repository URL' box, enter http://localhost:7001 and it will prompt you to install the addon, click on 'install' button and addon will be installed.

Click, the 'Discover' tab and in the sorts section you can see the 'Bollywood Thrillers' tab, clicking it you can see the bollywood thriller movies listed.

That's All! :) 