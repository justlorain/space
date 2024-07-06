---
title: "How to Build Your Own Chrome Extension Using ChatGPT"
description: "How to Build Your Own Chrome Extension Using ChatGPT"
publishDate: "4 Nov 2023"
tags: ["webdev", "ai", "programming", "tutorial"]
---

## Introduction

In my [previous article](https://dev.to/justlorain/i-created-a-chrome-extension-in-15-minutes-with-zero-front-end-knowledge-using-gpt-33df), I shared how I developed a Chrome extension using ChatGPT within 15 minutes, despite having no prior knowledge of frontend development. Recently, I completed another Chrome extension project with the help of ChatGPT. In this article, I want to summarize and share the entire development process and insights, hoping to provide some inspiration and assistance to you.

In this article, I will walk you through the process of developing the [TABSNAPSHOT](https://github.com/B1NARY-GR0UP/tabsnapshot) extension using ChatGPT. We will start from the basic requirements and move on to some key points to consider when developing with ChatGPT.

## Requirements

The idea for this extension came from a real-life scenario. I spend most of my time browsing web pages and PDFs. Chrome provides an excellent reading experience, but I often find myself opening specific pages related to a particular topic. For example, during my midterm exam preparation, I needed to open three pages (a lecture PDF and two article links) every time I opened Chrome. Using Chrome's built-in features, I had two options:

- Create a **folder** and save these three pages as bookmarks inside the folder. I could open all the pages at once using the "Open all bookmarks" option in the folder.
- Add these three pages to Chrome's **"Reading List"** and open them one by one from the Reading List.

However, I rejected both of these methods because:

- The process of creating a folder and adding pages to it is too **tedious**: creating the folder, adding a page to the folder, and saving it require too many steps. Besides, I usually use Chrome folders to collect web pages rather than storing these temporary pages I need to read.
- The **Reading List doesn't support adding PDFs**, and dealing with PDF pages separately would be a very cumbersome process.

So, I clearly identified the need for a tool that could **temporarily save one or multiple tab pages** and **open these pages with minimal actions**, similar to the `Open All Bookmarks` option.

## Development Approach

Unlike my previous GitHub Searcher project, I had no specific idea of how to implement this extension. I wasn't even sure if Chrome provided APIs to achieve these functionalities. After giving it some thought, I came up with a rough plan:

- Implement a button in the extension popup to save all currently open tabs as a "snapshot."
- Use the extension's popup window as the main control panel for the extension.
- Display all saved snapshots in the popup window. Clicking on a snapshot should open all the tabs saved in that snapshot.

With this plan in mind, I named the extension **TABSNAPSHOT** and proceeded to write the code with ChatGPT.

## Development Process

### Basic Functionality

Since I relied entirely on GPT to write the code, I communicated my requirements and approach clearly to GPT. However, due to GPT's nature, there were some challenges and issues that arose.

My initial prompt was as follows:

> I want to develop a Chrome extension called tabsnapshot. This extension should:
>
> 1. Save the URLs of all open tabs in the browser when the user clicks the "Create Snapshot" button in the extension's popup window (i.e., create a snapshot).
> 2. Display the saved snapshots as items in the popup window, with the snapshot's creation time as the item's name.
> 3. When the user clicks on a saved snapshot in the popup window, open the URLs saved when the snapshot was created.
>
> Please develop this extension and provide all necessary files.

However, the code generated based on this prompt had several issues:

- The `manifest_version` was set to **2**.
- Only the URL of the first open tab was saved in each snapshot, not all open tabs.
- Saved snapshots were **not persisted**. When I closed and reopened Chrome, the saved snapshots disappeared.

To address these issues, I provided additional instructions to GPT:

> I want to develop a Chrome extension called tabsnapshot. This extension should:
>
> 1. Save the URLs of all open tabs in the browser when the user clicks the "Create Snapshot" button in the extension's popup window (i.e., create a snapshot).
> 2. Display the saved snapshots as items in the popup window, with the snapshot's creation time as the item's name.
> 3. When the user clicks on a saved snapshot in the popup window, open the URLs saved when the snapshot was created.
>
> Please develop this extension and provide all necessary files.
>
> Additional instructions:
>
> 1. The `manifest_version` value should be set to 3.
> 2. Each snapshot item should save all open URLs, not just one URL. You can consider using an array or map for storage.
> 3. All snapshot items must be persisted. When I close and reopen the browser, I should see all previously saved snapshot items.

With these additional instructions, GPT generated code that met my requirements. I had the basic functionality of TABSNAPSHOT, allowing me to create snapshots, save them, and open them with a single click.

The initial UI for this basic version was quite simple:

![111](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b9itfjju5go5bzpr9u93.png)

Once the basic version was implemented, I had a foundation to build upon. I continued to enrich and optimize the features of TABSNAPSHOT.

### Delete Functionality and UI Enhancement

In the initial version, I only had the functionality to create snapshots but lacked the ability to delete them. I asked GPT to add the delete functionality:

> I need you to add snapshot deletion functionality to this extension. Without changing the existing functionality, add an "x" button next to each snapshot item in the popup window. Users can click this button to delete the corresponding saved snapshot. Please provide the updated code after adding the delete functionality.

With the code provided by GPT, the UI was updated as follows:


![222](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ujq1xg2h81nxvkbi4rjo.png)

Although the "x" buttons were functional, the UI was not aesthetically pleasing. I asked GPT to improve the style while keeping the delete functionality intact:

> Great, but the style of the delete button is not appealing. Can you make it similar to the "Create Snapshot" button and maintain the delete functionality?

The improved style of the delete button, along with other UI enhancements, looked like this:

![333](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/a5wlcnxkjf08l2a1o1qn.png)

We can see that the optimized delete button looks much better, even richer in style than the `Create Snapshot` button mentioned in the prompt, so we continue to let GPT optimize the UI, and the final UI looks like this:

![444](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n5b9kfjrzx8yl8uibdsi.png)

At this point, TABSNAPSHOT is fully operational, and all the current features are sufficient to address the pain points I mentioned in the requirements scenario.

But because of my OCD, we continue to add features and optimizations to TABSNAPSHOT.

### Rename and Open Snapshot

From the above UI, it can be seen that each snapshot entry is named after the time it was created. When there are many entries, it becomes difficult to distinguish which content each snapshot contains. Therefore, here we added the renaming functionality to TABSNAPSHOT:

> The first feature we are going to add is the renaming functionality. Please add a `rename` button to the left of the `delete` button with a style consistent with the `delete` button. When the user clicks the `rename` button, the original text of the snapshot entry will become editable. Users can rename the snapshot by modifying the text and confirming with the Enter key.

However, GPT did not provide a perfect solution in this case. When clicking the `Rename` button and modifying the text, the Enter key did not save the changes. Users had to click somewhere else in the popup window to save the changes. We had to use a few more prompts to guide GPT and make the necessary modifications to finally complete the development of the renaming functionality:

![555](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ck5hmihk2c1f78zrhoga.png)

Here, we also changed the way snapshots are opened to a specific button:

> Great! Now I would like you to modify the logic for opening saved snapshots. Currently, the logic is to click on the snapshot entry to open it. I want you to change the way snapshots are opened to clicking an `open` button. This `open` button should be located to the right of each snapshot entry. The style of a snapshot entry should be: Snapshot Name, Open button, Rename button, Delete button.

The final result looks like this:

![666](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0qvt1vgz3z9ag0u7jz86.png)

### Open All and Delete All

To further enhance the functionality of TABSNAPSHOT, I asked GPT to add "Open All" and "Delete All" buttons:

> Excellent. Now I want you to add two buttons to the right of the "Create Snapshot" button:
> - "Open All": Opens all saved snapshots.
> - "Delete All": Deletes all saved snapshots.

The updated UI with these features was as follows:

![777](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/j51k4y7e37l8j77i1tnj.png)

Now that we have completed all the major feature points, we will fine-tune some of the details of TABSNAPSHOT.

### Detail Optimizations

Optimizable details are as follows:

- If the tab contains a local PDF for browsing, it should automatically refresh after opening; otherwise, manual refresh might be necessary.
- Add a tab count for each snapshot entry.
- Simplify the default snapshot naming format (retain only month, day, hour, and minute).
- If multiple snapshots are created within the same minute, automatically add numbering to avoid duplicates.
- Snapshot preview.

When communicating with GPT through prompts, it's essential to describe your requirements clearly. You can make your prompts more vivid by providing examples:

> Next, we are going to add a tab count feature to the snapshots. When creating a snapshot entry, the number of tabs included in that snapshot will be displayed after the snapshot name. For example, a name for a snapshot containing three tabs would be `sample [3]`. Please note that this `[3]` should not be editable by the user through the rename button. You can place the snapshot entry name and tab count in separate elements but display them on the same line.

> Please add a logic check for snapshot creation. If multiple snapshots are created within the same minute, start numbering from the second snapshot created within that minute, following the order of creation. For example, the name of the first snapshot created at 15:53 on 11/1 would be 11/1 15:53, the second snapshot created at 15:53 on 11/1 would be 11/1 15:53 (2), and the third one would be 11/1 15:53 (3).

One thing worth mentioning is the snapshot preview feature. I have found it challenging to convey the exact effect I want to GPT, which might be why the implemented feature did not meet my expectations. Perhaps I haven't clearly defined what the preview feature should look like.

From a hover-based implementation to a click-based popup window:

> Now I want to add a preview feature to this plugin. Leave a space at the bottom of the popup window as the preview area. When the user hovers over a snapshot entry, the preview window will list all the links from the tabs included in that snapshot in the form of a list. Please implement this feature based on the above code.

> Now I want to add a preview feature to this plugin. When the user clicks on a snapshot entry, the browser will pop up a preview window (please note that this preview window is not the plugin's popup window). The preview window will list all the links from the tabs included in that snapshot in the form of a list. Please implement this feature based on the above code.

The final version, with all optimizations completed, is as follows. This is the v0.1.0 version released after open-sourcing:

- **Plugin UI**

![777](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/of2y4srgv7cq2jm4izbl.png)

- **Preview window UI**

![888](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h9a9s1jwlul8bcqt8i9b.png)

## Key Takeaways from Developing with ChatGPT

Here are some key points and lessons I learned during the development process with ChatGPT:

- Clearly define your **development goals and expectations**. Ensure you can articulate your requirements logically; otherwise, the responses from ChatGPT might not meet your needs.
- **Break down your project** into small functional points. Develop one feature at a time, starting with the basic functionality and then refining and optimizing it.
- **Long prompts** might cause ChatGPT to lose context. Consider using previous prompts to edit and submit or start a new chat to maintain context.
- Use **consistent terminology** and agreements with ChatGPT during the development process.

## Logo

You can also design an attractive logo for TABSNAPSHOT:

![999](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/alipd3n5pow6vbtvafms.png)

## Conclusion

This article has covered my entire journey of developing the TABSNAPSHOT extension with the help of ChatGPT. It includes the iterative thought process and the key points summarized at the end. I hope this experience can assist you in your projects.

As you can see, TABSNAPSHOT is still a very basic and simple extension with plenty of room for improvement. However, if you have a similar use case to mine, feel free to use TABSNAPSHOT to simplify your browsing experience.

If you find [TABSNAPSHOT](https://github.com/B1NARY-GR0UP/tabsnapshot) helpful or if you have suggestions for improvements, please feel free to **Star, Fork, and submit Pull Requests** !!!

## References

- https://github.com/B1NARY-GR0UP/tabsnapshot
- https://chat.openai.com/