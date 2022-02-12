---
url: "/projects/"
hidemeta: true
hideAboutAuthor: true
comments: false
---

# Personal Projects

## [Toggl Track Integration for Obsidian](https://github.com/mcndt/obsidian-toggl-integration) (2021-today)

<div style="display: flex">
  <img src="https://img.shields.io/github/v/tag/mcndt/obsidian-toggl-integration">
  <img style="margin-left: 0.5em" src="https://img.shields.io/github/downloads/mcndt/obsidian-toggl-integration/total">
  <a href="https://github.com/mcndt/obsidian-toggl-integration/stargazers" style="box-shadow: 0 0">
    <img style="margin-left: .5em" src="https://img.shields.io/github/stars/mcndt/obsidian-toggl-integration.svg?style=social&label=Star&maxAge=2592000">
  </a>
</div>

[GitHub](https://github.com/mcndt/obsidian-toggl-integration) 

As an avid user of both the Obsidian notes app and the Toggl time tracking service, I develop and maintain an open-source plugin integration Toggl into Obsidian, downloaded by over 2,700 users to date. 

The plugin is self-documented using TypeScript and makes use of the official [Obsidian API](https://github.com/obsidianmd/obsidian-api) and The [Toggl and Toggl Reports API](https://github.com/toggl/toggl_api_docs). Custom views are written in the Svelte 3 framework.

![Demo](/media/obsidian-toggl-demo.gif)

Click [here](obsidian://show-plugin?id=obsidian-toggl-integration) to open the plugin in your Obsidian client.

## [blockcolors.app](https://blockcolors.app) (2021-today)

I started [blockcolors.app](https://blockcolors.app) as a creative outlet during my thesis year in university. The app lets users discover new combinations of building blocks in Minecraft, save their favorite palettes, and share palettes with friends using share links. 

The representative colors of each Minecraft texture were extracted in Python using the PIL and skimages libraries and an algorithm I wrote based on the [median cut algorithm](https://en.wikipedia.org/wiki/Median_cut) used in the GIF image standard.

The web app is built using SvelteKit and Tailwind, and is deployed with [Cloudflare](https://pages.cloudflare.com/) for fast worldwide access at the edge As a Jamstack app, all logic runs either client side or is deployed as a Coudflare Worker.

![Palette builder](/media/blockcolorsapp/blockcolors1.png)
![Palette viewer](/media/blockcolorsapp/blockcolors2.png)
![Palette gallery](/media/blockcolorsapp/blockcolors3.png)