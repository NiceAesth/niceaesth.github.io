---
title: Extracting iOS Ringtones from .ipsw
date: 2023-09-21 02:09:00 +03:00
categories: [Technology, iOS]
tags: [iOS, ringtones]
---

## Backstory

This will be a fairly short post, as this process is fairly straightforward.

Recently, Apple has added a selection of new ringtones to iOS 17. These were very cool! Too bad you can't obtain the files to them. Or can you?

## How to do it

1. Download the appropriate `.ipsw` file. You may use a service such as [IPSWBeta.dev](https://ipswbeta.dev/ios/)
    > Note: I am not affiliated with IPSWBeta.dev and I am not responsible for their actions.
    {: .prompt-warning }
2. Open the `.ipsw` file using `7-Zip` (right click and open archive).
3. You will see multiple `.dmg` files inside. Sort by size, and extract the largest one.
4. Open the `.dmg` file using whatever method you prefer. I chose to just do this on macOS as it was the easiest.
5. Navigate to the `Library/Ringtones/` folder.
6. ???
7. Profit!

Additionally, you may obtain the wallpapers which are stored in the appropriately named `Wallpaper` folder.

## For those too lazy to do this

I have uploaded my dump of the iOS 17 ringtones [here](https://up.aesth.dev/J3pgng8h.zip). I do not guarantee that this link will remain valid forever, but the instructions above should work in case it does go away.
