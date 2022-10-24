---
author: "Hugo Authors"
title: "How I Built This Website"
date: 2022-10-20T12:00:06+09:00
description: "Guide to building a free website with HUGO, GitHub, and Netlify."
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Jacob Kollasch
authorEmoji: By
pinned: true
tags: 
- hugo
- github
- netlify
---
This article is a guide for building a website using HUGO, and how to host it on the internet using GitHub and Netlify for FREE. 

I decided to write post this because the guides I came across when I built this website all seemed to leave out key details. I spent several days researching, and troubleshooting this process. 

My goal with this guide is to walk you through standing up a website template, and show you how to make it your own. So you can get right to sharing your work!

How it works from a high level:
1. Create, edit and test website on your local computer using HUGO.
2. Push your website files to a GitHub repository.
3. Netlify will monitor the GitHub repository for changes. When changes are detected it will automatically rebuild the website, and publish it on the internet using their CDN.

## Why I Chose Hugo and Netlify

I have wanted to build my own website to demonstrate what I was learning in my IT career to myself, peers, fellow learners, and potential employers. Inspired by sites like [routerfreak.com](https://www.routerfreak.com) and [network lessons](https://www.networklessons.com). 

At first, I thought building a website would not be worth the trouble. Would it would require hosting a linux server running apache, or a Windows server running ISIS? Would I have to run a server at my house and open up ports in my firewall? Could I afford to host it in a cloud service like linode? How would I properly secure the server, manage patching, and backups? Am I going to have to learn HTML, CSS, and Javascript just to stand up a website? I wasn't looking for a full time job maintaining my personal website. 

I thought about using a template website like wordpress, or square space, but that felt too easy. I didn't want to have to work inside of a web browser, or pay a monthly fee to host my website. I was looking for something in between hosting my own web server, and essentially filling out an online form. Then I discovered the magic of using HUGO with Netlify, and I knew I found something worth diving into.

### What is HUGO and Netlify
#### HUGO

HUGO is a website generator that has tons of community built quality themes to help get you started. One thing I love most about HUGO is it's FAST! HUGO sites are also very easy to work with.  Once you start to learn about how the configuration files work, and make your first post using markdown you will appreciate the simplicity and flexibility it offers.

#### Netlify

Netlify is CDN (Content Delivery Network) that will build, deploy, and host your website on the internet. Simply link your GitHub repository containing your HUGO site to your Netlify account, and it will take care of the rest. The best part of Netlify is you get a free 100GB of traffic per month. This is more then enough for a blog type website. Netlify also simplifies the certificate management process by using Let's Encrypt, so your page will be available using HTTPS.

## Installing HUGO

HUGO can be installed on windows, macOS, or linux form the command line using a package manager. 

A package manger is basically an app store for your command line. We can search, install, and remove applications by running simple commands. If you don't know how to use a package manager there is plenty of good information out there to get you started. I have listed the install commands for the recommended package managers to install HUGO. If you get stuck installing HUGO check out the official [documentation](https://gohugo.io/getting-started/installing/).
### Windows (Chocolatey)

`choco install hugo -confirm`

### macOS (Homebrew)

`brew install hugo`

### linux (apt)

`sudo apt install hugo`


## Creating a HUGO Site

You will create your HUGO site on your local computer, and download a theme. I am going to use the ZZO theme that I used to build my website. ZZO has good [documentation](https://zzo-docs.vercel.app/zzo/changelog) that can help you customize your page, and it has a clean simple design. Check out other HUGO themes [here](https://themes.gohugo.io/). 

Create your website using HUGO from the command line.

`hugo new site example-website`

You can replace "example-website" with the name of your website. Don't type in a full URL like www.example-website.com. Just use the root domain. This will create a directory with default files needed for HUGO to build a website.

Now we need to change directory to our new website folder. 

```
cd example-website
```

Next we will use git to initialize the directory and download the ZZO theme as a submodule.
```
git init
git submodule add https://github.com/zzossig/hugo-theme-zzo.git themes/zzo
```

### Using the ExampleSite

Now that we have installed our theme we can use the example site as a base to get started. 

The easiest way to do this is to copy the example site folder contents into the root of our website directory. This will replace the empty HUGO generated folders with the example site folders.

Navigate to the ```/example-website/themes/zzo/exampleSite``` directory, and copy the ```config```, ```content```, ```resources```, and ```static``` folders. Paste them to the root directory of your website ```/example-website/```. You should be asked if you want to replace the folders of the same name. Select "yes". 

Next you want to delete the ```config.toml``` file in the root directory of your website that was created when we first built our website. 

```config.toml``` is an important config file that HUGO uses to build your website. By default, HUGO looks for ```config.toml``` in one of two places. 
1. Root directory of the website ```/example-website/```
2. A directory called "_default found under ```/example-website/config/```


When we copied the example site into the websites root directory we added a new ```config.toml``` file to the ```config/_default``` folder, so if we don't delete the default ```config.toml``` file HUGO will use the default file and your theme won't be used. 

### Comment out the BASE URL in config.toml

One important change we need to make to the config of the example site is to comment out the base URL in ```config.toml```. When you first build a website with Netlify they will give you a domain, and you can later go and configure it here, or if you use a custom domain you will want to add it to the baseURL line.

For now we will just comment it out, so the site can be built. It shouldn't cause any issues with the deployment.

### Testing the website locally

Okay now we are ready to build our website locally on our computer. Go back to your command line and make sure you are in the root directory of your website ```/example-website/```. Then run the following command:

```hugo server```

HUGO will build your website and make it available to your computers loopback address using an ephemeral port. By default it will try to use port 1313, but if you already have HUGO running it will use another port. Just look at the end of the ```hugo server``` output to see the URL you can use to view your new website on your local machine. For example:

{{< highlight shell >}}
jacob@pop-os:~/Documents/GitHub/example-website$ hugo server
port 1313 already in use, attempting to use an available port
Start building sites … 
hugo v0.92.2+extended linux/amd64 BuildDate=2022-02-23T16:47:50Z VendorInfo=ubuntu:0.92.2-1
WARN 2022/10/22 11:53:14 The "twitter_simple" shortcode will soon require two named parameters: user and id. See "/home/jacob/Documents/GitHub/example-website/content/en/posts/rich-content.md:35:1"

                   | EN  | KO   
-------------------+-----+------
  Pages            | 132 |  28  
  Paginator pages  |   3 |   0  
  Non-page files   |   3 |   0  
  Static files     | 127 | 127  
  Processed images |   0 |   0  
  Aliases          |  36 |  10  
  Sitemaps         |   2 |   1  
  Cleaned          |   0 |   0  

Built in 3031 ms
Watching for changes in /home/jacob/Documents/GitHub/example-website/{archetypes,content,data,layouts,static,themes}
Watching for config changes in /home/jacob/Documents/GitHub/example-website/config/_default
Environment: "development"
Serving pages from memory
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:45755/ (bind address 127.0.0.1)
Press Ctrl+C to stop
{{< /highlight >}}

This is roughly the output you received after running "hugo server". Look at the second to last line. This will tell you how to access your website. I already had HUGO running using the default 1313 port, so HUGO used port 45755. To access my website I would enter "localhost:45755" into my browser. Now you should see ZZO example website running locally on your computer.

Before you make changes and start customizing your website let's get it online, so you can see understand the process.

put example image here

## Creating a GitHub Repository

At this point, we have created our website using the example site template. Now you will need to upload the the website to GitHub and create a repository. We will use GitHub Desktop to do so in this guide.

### GitHub Desktop Vs. git

GitHub desktop is a GUI application that allows you to work with code repositories hosted on GitHub without having to learn how to use the git version control system. Git is a version control application that can be used from the command line to work with code repositories.

Learning how to use git from the command line could be a lengthy writeup on it's own. [GitHub Docs](https://docs.github.com/en/get-started) is a great resource if your interested. Our goal today is to get a website running not learn git, so I would suggest doing it the easy way with GitHub Desktop for now.

You can install GitHub Desktop for Windows and macOS [here](https://desktop.github.com/)

### Installing GitHub Desktop on linux
Installing GitHub Desktop isn't as easy as downloading an installer on linux, but I was able to get it done using the following commands provided by GitHub user [berkorbay](https://gist.github.com/berkorbay/6feda478a00b0432d13f1fc0a50467f1).

{{< highlight shell >}}
sudo wget https://github.com/shiftkey/desktop/releases/download/release-2.9.3-linux3/GitHubDesktop-linux-2.9.3-linux3.deb
sudo apt-get install gdebi-core 
sudo gdebi GitHubDesktop-linux-2.9.3-linux3.deb
{{< /highlight >}}

### Adding a Repository Using GitHub Desktop

At this point if you don't have a GitHub account you will need to go create one. Then you will need to sign into your GitHub account in GitHub Desktop. After you have done that you are ready to add a repository to your GitHub account.

In GitHub Desktop, go to file > add local repository > select the path of your website > click "Add repository"

In the left hand pane you will see the file changes you have made. In the bottom left there is a required summary field. Enter a useful summary. For example, "creating website repository". Click "Commit to master".

Then in the top of the page click "Publish Repository".

Now go to your GitHub account online and verify the repository has been created.

## Configuring Netlify

### Linking GitHub Repository to Netlify
At this point you will need create an account on Netlify, and link your GitHub account. This is very straight forward, and Netlify will guide you through the process.

### Configure Netlify to Build Using HUGO
Create a new site from the overview page of your netlify account. Select "Import an Existing Account". Chose GitHub as your git provider. Then select the repository containing your website.

Now you will be prompted to configure your website "Build Settings". This is where we will tell Netlify that our website should be built using HUGO. Fill out the "Basic Build Settings" as below.

### Your Site is Live!
At this point your website should be live on the internet. Pretty slick!

## Creating a Custom Domain

After you create your website on Netlify you will be given a URL that anyone can type into their web browser and get to your page. However, this URL will be auto generated and will be something like [iridescent-muffin-697134.netlify.app](iridescent-muffin-697134.netlify.app).

This isn't very nice if you are hoping to put your website on your resume, or have people remember your site and be able to type it into an address bar easily. If you want the URL of your website to resemble the actual name of your website. You will have to use a custom domain.

Using a custom domain for your website is NOT FREE. You will need to purchase a domain or bring your own. Netlify makes it really easy to buy a domain. For example, I bought the domain for my website for $12.99 a year. Personally I think it's worth it because it make your websites address memorable and readable.

## How to Update your Website

The best part about how we built this website is you can work on it from any computer with HUGO and GitHub desktop or Git installed. I would highly recommend using VSCODE to make things easier on yourself, but it's not necessary. 

Here is how you will do it from a high level:

1. Clone your website repository from GitHub to you local machine. This is easily done with GitHub Desktop. 
2. Edit the files of your website, or create a new post. 
3. Commit and publish your changes to your repository just like we did earlier using GitHub Desktop. 
4. You can view the build status same as before on Netlify, and finally your changes should be live.

## Customizing your Website
Coming Soon...

I will update this section later with some more information on making your website your own. In the meantime check out the zzo documentation, and play around with the config files of your website to see what you can come up with.

# Conclusion
Have fun customizing your page, exploring new themes, and don't be afraid to break your site. I did that several times myself when learning this process. You will learn more from fixing your mistakes than you ever could reading a guide like this. 

Good luck, and I hope this can help someone avoid the problems I ran into!