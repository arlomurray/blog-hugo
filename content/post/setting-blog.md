+++
date = "2017-04-12T12:53:14+09:00"
title = "Setting up a blog with Github Pages and Hugo"
+++

As the topic is still fresh on my mind from creating this blog, I thought I would write a quick post on how others can create their own.

## What is Hugo and Github Pages
![Hugo](https://github.com/arlomurray/blog-hugo/images/hugo.png)
Hugo is a static website generator, which makes it ideal for creating things like blogs, personal websites, or any other site that does not require dynamic content. It is very simple to install and configure on any operating system, be hosted anywhere, is incredibly fast, *and* it is written in the Go programming language. Being quite a avid fan/user of Go, I was naturally drawn to the platform. It also seemed easier to use than Wordpress, which I've had my fair share of bouts with in the past. On top of all that, Hugo is a free platform.

However, since Hugo only generates static pages, the site needs to be hosted by another service. (Hugo actually can serve websites, but it does not seem to be a popular option.) In my case, this is where Github Pages comes into play. Anyone that has used Github or follows blogs of programmers have probably heard of Github Pages. Hosting a site on GP is also free, plus the site can have a custom domain and be secured with HTTPS. Full Disclosure: I've heard mixed reviews about Github Pages, but decided to give it a shot.

Being familiar with git and a text editor, I thought this would be a breeze. Luckily, it was!

*I'm going to assume that anyone following this already has git configured on their computer.*

## Installing Hugo
**For Windows Users:**

1. Create a new folder in your C drive and name it `C:\Hugo`
2. In the Hugo folder, create two subfolders named `bin` and `Sites`
3. Download the Windows Hugo package from [here](https://github.com/spf13/hugo/releases)
4. After downloading, extract the files into the newly created `bin` folder
5. Rename the executable `hugo.exe`
6. Using the command line, `cd` into `C:\Hugo\bin` and set the path by entering `set PATH=%PATH%;C:\Hugo\bin`
7. Restart the terminal for the changes to take place
8. Check to see it was successful by entering the command `hugo help`


**For Mac Users:**

*The easiest way to install Hugo on a Mac, is by utilizing Homebrew. Please install it before hand.*

1. Run the command `brew update` in the terminal (if you had already had Homebrew installed on your machine)
2. Run the command `brew install hugo`
3. Verify that Hugo was installed successfully by running `hugo help` (you may need to restart the terminal first)

## Creating a New Site
Setting up a new site with Hugo is pretty straightforward. If you are using Windows, navigate to the subfolder `Sites` in `C:\Hugo` using the command line. If you are using MacOS, pick a convenient place and `cd` into it. I chose to have my site live in my `src` directory.

Enter the command `hugo new site blog`. Substitute `blog` with whatever you want to name your site. This should create a scaffold in whatever directory you ran the previous command in.

Next, you can install a theme of your choice. For this example, I will use the [theme](http://themes.gohugo.io/hugo-theme-cactus-plus/) that I'm currently using on this site. To install a theme, `cd` into the `themes` folder inside of your site. Run the command `git clone https://github.com/nodejh/hugo-theme-cactus-plus.git`, substituting the link for the Git repository to the theme you have chosen. Take a look at the `config.toml` file in the theme's repository for all the specifications you can modify. For example, in `hugo-theme-cactus-plus` I can specify my social media handlers, the title of the site, bio, etc. Whether you specify those or not is totally up to you. However, you do need to specify the `baseurl`, `themesDir`, and `theme`. For `baseurl`, I put `/` as the theme did not render properly if I put the url to my site. For `themesDir`, I put `themes/`, and for `theme` I put the name of the theme as it is in the `themes` folder. In my case, that was `hugo-theme-cactus-plus`.

If all went well, you should be able to enter the command `hugo server -w` and go to `localhost:1313` to see your new site. It may take a minute for everything to render properly.

## Setting Up New Repositories
To host our new site using Github Pages, we actually have to create two repositories on our Github profile. One will host all of the Hugo content while the other will host the site and be placed inside the other repository.

Create the first repository and name it `<USERNAME>.github.io` and check the "Initialize this repository with a README" box. Then create the second repository and name it anything you like (preferably it will be what you named your site folder). `cd` into your project folder and enter these commands:
```
$ git init

$ git remote add origin git@github.com:<USERNAME>/blog.git
```
We should now be able to serve our site locally. Enter `hugo server -t hugo-theme-cactus-plus` in the terminal, substituting the theme with the one you chose for your site. You should see your site if you go to `localhost:1313`. Stop the server. Remove the public folder that was generated with `rm -rf public`. Then enter this command:
```
$ git submodule add -b master git@github.com:<USERNAME>/<USERNAME>.github.io.git public
```
This command should have added the static site into the `public` folder.

Before deploying the site, add all the files from the main `blog` repository and push it up to Github.
```
$ git add .

$ git push -u origin master
```
Finally, to deploy the site first set the theme, add all the files from the `public` folder, make a commit, and push up to the `<USERNAME>.github.io` repository.
```
$ hugo -t hugo-theme-cactus-plus
$ cd public
$ git add .
$ git commit -m "deploy site"
$ git push origin master
```
There may be a delay, but after a few moments you should be able to see your new site at `<USERNAME>.github.io`!

## Automate Deployment
Optionally, you could create a shell script to deploy the site for you instead of having to do the last part every time you change the site.
To do so, create a new file called `deploy.sh` and add the following to it (this is straight from the Hugo tutorial):
```
#!/bin/bash

echo -e "\033[0;32mDeploying updates to GitHub...\033[0m"

# Build the project.
hugo # if using a theme, replace by `hugo -t <yourtheme>`

# Go To Public folder
cd public
# Add changes to git.
git add -A

# Commit changes.
msg="rebuilding site `date`"
if [ $# -eq 1 ]
  then msg="$1"
fi
git commit -m "$msg"

# Push source and build repos.
git push origin master

# Come Back
cd ..
```
Make the following script an executable by entering `chmod +x deploy.sh`. Now to deploy your site, all you would need to do is enter `./deploy.sh` in the terminal. Note: This script only adds changes from the `public` folder, so if had modified something in the `blog` folder you would manually need to push that up to Github.
