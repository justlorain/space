---
title: "Created a Chrome Extension in 15 Minutes Using GPT"
description: "I Created a Chrome Extension in 15 Minutes with Zero Front-End Knowledge Using GPT"
publishDate: "10 Sep 2023"
tags: ["webdev", "ai", "programming", "tutorial"]
---

## Preface

When reading, authors often mention various projects or related materials, and sometimes it's just plain text without corresponding links. If I want to learn more about these projects, I have to search for them myself, either through Google or other means. One common thing I often do is:

- Copy the name of a project.
- Open a new tab.
- Visit GitHub.
- Paste and search.

Perhaps because I've been repeating this process too frequently recently, I came up with the idea to develop a plugin to simplify this process.

So, a few minutes later, I created my first Chrome extension, [GitHub Searcher (GHS)](https://github.com/justlorain/ghs).

## Expected Outcome

The desired outcome for the plugin is very simple, or one could say rudimentary: **search for the text I've selected on GitHub, and be able to choose whether to search in the current tab or open a new tab**. With this feature, I can simplify the four steps mentioned earlier into just two:

- Select the content I want to search for.
- Right-click and choose whether to search in the current tab or open a new tab.

## Development Approach

After trying to search for several repositories on GitHub, I found that the URL for searching repositories on GitHub is:

```html
github.com/search?q=test&type=repositories
```

In this URL, the `q` parameter corresponds to the search query content, and the default `type` is `repositories`. Therefore, to achieve the desired outcome, it becomes very simple. I just need to find a way to get the selected text, then concatenate it and visit the URL.

## Development Process

With the basic idea in mind, it's time to start development. Since I primarily use Chrome, I decided to implement it as a Chrome extension.

I quickly went through the [Development basics](https://developer.chrome.com/docs/extensions/mv3/getstarted/development-basics/) provided by the official Chrome documentation to understand the project structure and necessary files for a Chrome extension. However, my knowledge of front-end development is limited to a few common HTML tags. This is where I relied on my trusty friend GPT.

I organized my earlier thoughts into language that GPT could understand and fed them to it, and then... GPT generated all the code for me.

Initially, I didn't have very high expectations. I thought it would generate some code samples using Chrome API, which I could then modify to make it work. But when I copied the code generated by GPT into my project and loaded the extension in Chrome, the desired functionality was fully implemented!

My prompts to GPT were quite simple, as follows:

> **Me:** What's the operation called when you select a portion of text in a web page by pressing and dragging the mouse?
>
> **GPT:** In a web browser, the operation of selecting a portion of text by pressing and dragging the left mouse button is typically referred to as "text selection" or "text dragging." This operation allows users to highlight text on a web page for purposes such as copying, cutting, or performing other actions like searching or translating. It's a common interaction when reading web pages, copying information, or quoting text.

> **Me:** What's the operation called when you right-click with the mouse in a web browser to open a menu?
>
> **GPT:** In a web browser, the operation of opening a menu by right-clicking with the mouse is usually called "context menu" operation, sometimes referred to as the "right-click menu." This menu typically contains options related to the element or area you right-clicked on, allowing you to perform various actions like opening links, saving images, copying text, and more. The content of this context menu can vary depending on what you right-clicked and your browser's settings. It provides a convenient way to access common functions quickly.

> **Me:** In some Chrome extensions, the right-click menu often has a submenu, such as "Image Helper," which contains various supported actions like "Extract Images from this page" and "Download Images from this page." Is there an official term for these?
>
> **GPT:** In Chrome extensions, these submenus within the right-click menu, which contain multiple related actions, are typically referred to as "context menu submenus" or "right-click submenus." These submenus allow extension developers to organize different functionalities supported by the extension into a larger group of options, providing more functionality choices and an improved user experience. In this submenu, each option usually represents a specific operation or feature supported by the extension. In the example you mentioned, the "Image Helper" submenu includes actions like "Extract Images from this page" and "Download Images from this page."
>
> It's important to note that there isn't an official standard term for this, and different extension developers may use different terminology to describe this functionality. However, in general, the purpose of such submenus is to organize and present multiple related action options.

> **Me:** I want to develop a Chrome extension called "ghs" (GitHub Searcher). The usage of this extension is as follows:
>
> 1. The user selects some words or sentences on a web page using the mouse, which is the "text selection" operation.
> 2. The user right-clicks with the mouse to open a context menu and selects a submenu named "GitHub Searcher." This submenu contains two options: "Current Tab" and "New Tab."
> 3. After selecting text, clicking on these two options will open the URL `github.com/search?q={{ sample }}&type=repositories` in either the current tab or a new tab, where `{{ }}` represents the selected text.
>
> Here's an actual use case: While browsing a web page, I select the word "helloworld," then right-click and choose the "New Tab" option from the "GitHub Searcher" submenu. The browser should then open a new tab and visit the URL `github.com/search?q=helloworld&type=repositories`.
>
> Now, I need your help to develop this extension. Please list each step and add necessary comments to the code to help me understand it.
>
> **GPT:** Developing a Chrome browser extension involves several steps. I'll list each step and add comments to the code to help you understand it.
>
> **Step 1: Create the Extension Directory and Manifest File**
>
> First, you need to create a local directory for your extension and create a manifest file (`manifest.json`) within that directory. The manifest file is used to define the extension's basic information, permissions, and functionality.
>
> ```json
> // manifest.json
> 
> {
>   "manifest_version": 3,
>   "name": "GitHub Searcher",
>   "version": "1.0",
>   "description": "Search GitHub repositories from selected text.",
>   "permissions": ["contextMenus"],
>   "icons": {
>     "48": "icon.png"
>   },
>   "background": {
>     "service_worker": "background.js"
>   },
>   "permissions": ["activeTab", "contextMenus"],
>   "action": {
>     "default_popup": "popup.html",
>     "default_icon": {
>       "16": "icon.png",
>       "48": "icon.png",
>       "128": "icon.png"
>     }
>   }
> }
> ```
>
> **Step 2: Create the Extension Icon**
>
> Add the extension icon (`icon.png`) to the extension directory. This icon represents your extension.
>
> **Step 3: Create the Context Menu**
>
> In the `background.js` file, create the context menu so that users can right-click selected text to open the extension's submenu. Here's an example `background.js` file:
>
> ```js
> // background.js
> 
> chrome.contextMenus.create({
>   id: "githubSearcherContextMenu",
>   title: "GitHub Searcher",
>   contexts: ["selection"]
> });
> 
> chrome.contextMenus.create({
>   id: "githubSearcherCurrentTab",
>   title: "Current Tab",
>   parentId: "githubSearcherContextMenu",
>   contexts: ["selection"]
> });
> 
> chrome.contextMenus.create({
>   id: "githubSearcherNewTab",
>   title: "New Tab",
>   parentId: "githubSearcherContextMenu",
>   contexts: ["selection"]
> });
> 
> // Add click event listeners
> chrome.contextMenus.onClicked.addListener((info, tab) => {
>   if (info.menuItemId === "githubSearcherCurrentTab") {
>     openGitHubSearchPage(info.selectionText, false);
>   } else if (info.menuItemId === "githubSearcherNewTab") {
>     openGitHubSearchPage(info.selectionText, true);
>   }
> });
> 
> // Function to open the GitHub search page
> function openGitHubSearchPage(selectedText, newTab) {
> 	const searchUrl = `https://github.com/search?q=${encodeURIComponent(selectedText)}&type=repositories`;
>     if (newTab) {
> 		chrome.tabs.create({ url: searchUrl });
> 	} else {
> 		chrome.tabs.update({ url: searchUrl });
> 	}
> }
> ```
>
>   **Step 4: Create the Extension Popup Page**
>
>   Create a popup page (`popup.html`) that will display information or settings when the user clicks the extension icon. You can customize this page as needed.
>
>   **Step 5: Load the Extension**
>
>   Finally, load your extension in the Chrome browser:
>
>   1. Open the Chrome browser and go to the extensions management page (chrome://extensions/).
>   2. Enable "Developer mode."
>   3. Click the "Load unpacked" button and select your extension directory.
>
>   Once you've completed these steps, your extension should be able to display the "GitHub Searcher" submenu when the user selects text and right-clicks, and it should open the corresponding GitHub search page when "Current Tab" or "New Tab" is selected.
>
>   Please note that this is a basic example, and you can further expand and customize the extension's interface according to your needs. Also, ensure that you follow the Chrome Web Store's publishing guidelines and best practices. Once successfully published, users can install and use your extension from the Chrome Web Store.

You can see that the first three steps were primarily about aligning terminology and concepts with GPT. Only in the fourth step did I start combining these terms and my own ideas to have GPT generate the actual code.

As I mentioned earlier, the code provided by GPT was essentially correct and achieved the desired functionality when installed, but there were some flaws that needed to be addressed, such as the absence of `popup.html` and a reminder to set the icon. However, these were minor issues, and after designing a nice-looking icon and adding supplementary files like `popup.html` (optional), README, LICENSE, etc., our extension was complete :)

![GitHub Searcher Icon](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6otqp3vp52zuexnl7dqc.png)

## Conclusion

The above is the complete process of developing a Chrome extension from scratch with the help of GPT. A significant reason for the smooth development was that the functionality of this extension was quite simple, hhhh, but it certainly saved me some time.

Due to the complexity of publishing extensions in the Chrome Web Store, I didn't go through that process. If you're interested in this extension, you can download the source code and manually load it into your Chrome browser. Of course, if you find this extension useful, feel free to give the project a Star!

## References

- https://github.com/justlorain/ghs
- https://developer.chrome.com/docs/extensions/mv3/getstarted/development-basics/