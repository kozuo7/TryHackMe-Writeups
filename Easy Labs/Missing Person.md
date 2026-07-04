# TryHackMe Missing Person Write-Up

## Overview

Missing Person was a TryHackMe OSINT-style room built around a `.zip` file containing two JPG images:

```text
food.jpg
MotoGP.jpg
```

The scenario said:

```text
My friend went on holiday in 2025 and shared some photos, but I haven’t heard from him since. Can you help me track him down for the police report?
```

That `2025` detail ended up being important later because, at first, I focused too hard on the date from one of the images I found online.

This room was different from the labs I had been doing before. Instead of web vulnerabilities, terminals, and tool usage, it was more about image analysis, reverse image searching, metadata, social media digging, and connecting small OSINT clues together.

---

## ZIP Inspection

The room started with a `.zip` file containing two images:

```text
food.jpg
MotoGP.jpg
```

I opened both files first to see what was visually shown.

`MotoGP.jpg` showed a MotoGP race scene with a large `Pertamina` sign in the background and one racer visible.

`food.jpg` showed an outdoor area with a lot of wooden seating and a stall or restaurant on the right side selling food.

After that, I checked the file properties for both images to see if there was any useful metadata.

In Windows File Explorer, `food.jpg` showed a date taken of:

```text
10/5/2025 7:55 PM
```

`MotoGP.jpg` showed a date taken of:

```text
10/5/2025 12:33 PM
```

At this point, I knew both images were probably connected to the same trip day, but I did not know yet how important the exact metadata timestamp would become.

---

## Identifying the Circuit

The first question asked for the commercial name of the circuit.

The biggest clue was the large `Pertamina` sign in the MotoGP image.

I looked up what Pertamina was and found that it is Indonesia’s state-owned oil and natural gas corporation.

The important part from that search was the country:

```text
Indonesia
```

From there, I searched for MotoGP circuits in Indonesia and found:

```text
Pertamina Mandalika International Circuit
```

I tried that first, but it was incorrect.

So I pivoted and looked for alternative names. That led me to the full commercial name:

```text
Pertamina Mandalika International Street Circuit
```

That answer worked.

The key lesson here was that I had the right place, but the room wanted the exact commercial name.

---

## Reverse Image Searching MotoGP.jpg

I reverse image searched `MotoGP.jpg`, while trying not to click on other TryHackMe write-ups.

Eventually, I found this article:

```text
https://www.thejakartapost.com/business/2022/02/10/pertamina-named-title-sponsor-of-indonesian-grand-prix
```

The article contained the exact same image with a caption under it.

The caption said the image showed a motorcyclist testing the Pertamina Mandalika International Street Circuit in Lombok, West Nusa Tenggara, on November 8, 2021, ahead of the MotoGP race.

I noticed the `2021` date and initially thought I had found the correct event.

The article talked about the race that was being prepared for, which was the March 18 to March 20, 2022 MotoGP event.

So when the room asked when the event took place, I first tried:

```text
18-20/03/2022
```

That was incorrect.

This is where I realized I had made a bad assumption. The scenario specifically said the friend went on holiday in `2025`.

So even though the article had the exact same image, the event date from the article was not the answer the room wanted.

After that, I searched for MotoGP races in Indonesia during 2025 and found that the 2025 Indonesian MotoGP event took place on:

```text
October 03-05 2025
```

That answer worked.

---

## Identifying the Restaurant

The next question asked for the name of the restaurant.

For this part, I reverse image searched `food.jpg`.

That quickly led me to the restaurant:

```text
Cantina Mexicana
```

The location was in Indonesia, which matched the direction the investigation was already going.

---

## Finding the Exact Time the Photo Was Taken

The room also asked for the time the food photo was taken.

At first, I used the time shown in Windows File Explorer:

```text
7:55 PM
```

The required format was:

```text
HH:MM:SS
```

My first attempt was:

```text
07:55:00
```

That was incorrect.

Then I realized it most likely wanted 24-hour time, so I converted it to:

```text
19:55:00
```

That was also incorrect.

At that point, I got stuck for a bit. After doing some research, I found out that Windows File Explorer hides the seconds in the `Date taken` and `Date modified` metadata fields.

There was a way to extract the raw EXIF metadata with PowerShell, but I used an easier third-party metadata viewer instead.

I used:

```text
exifmeta.com
```

I dragged and dropped `food.jpg` into the EXIF metadata extractor.

The page showed different metadata tabs:

```text
FILE
JFIF
EXIF
COMPOSITE
RAW
```

I only needed the EXIF tab.

Inside the EXIF data, I found the `DateTimeOriginal` field. That showed the precise time:

```text
19:55:30
```

I submitted that and it worked.

This was probably the biggest new technical thing I learned from the room: Windows may show image metadata, but it does not always show the full raw EXIF value.

---

## After-Party Clue

After the first few questions, the room gave another message:

```text
He sent me a message, this is the last I heard from him: ”Went to this cool MotoGP after party, and became friends with one of the local DJs who played that night. We’re going to visit a cave tomorrow.”
```

From this, I took note of a few important clues:

```text
MotoGP after party
local DJ
cave tomorrow
```

The goal now was to figure out where the after-party happened, who the local DJ was, and what cave he may have taken tourists to.

---

## Finding the After-Party Location

I searched Google for:

```text
motogp pertamina grand prix of indonesia 2025 afterpartys
```

This gave me two main results:

```text
Mandalika Beach Club
Surfers Bar Kuta
```

I looked through both.

At first, I assumed Mandalika Beach Club was probably not the right vibe. It looked more resort-like, and it did not seem as likely that this random missing person would end up there and become friends with a local DJ.

So I focused on:

```text
Surfers Bar Kuta
```

I looked it up on Google Maps and found the full address there.

When submitting the answer, I had some input errors at first because copying and pasting the full address did not work.

The fix was removing `Indonesia` from the address line.

After that, the answer worked.

---

## Finding the DJ’s Stage Name

The next question asked for the DJ’s stage name.

The answer format showed a pattern like:

```text
**** *****
```

That helped because the first word was four characters, so it probably was not a DJ name that started with `DJ`, like `DJ Snake`.

I went to the actual Surfers Bar Instagram page and found their promotional post for the MotoGP after-party.

The post listed three DJs and their nationalities.

Since the message said the missing person became friends with a local DJ, I focused on the Indonesian DJ listed in the set.

The Indonesian DJ was:

```text
Bong Leleh
```

That answer worked.

---

## Searching the DJ’s Accounts

The next question asked:

```text
After digging into the DJ's other online accounts, what cave does he take tourists to?
```

So I started digging through Bong Leleh’s online accounts.

I first looked through his Instagram and checked his posts, especially around October 6, 2025, to see if there were any signs of a cave or tour.

I did not find anything useful there.

Then I found his Facebook account under:

```text
DJ Bong
```

I searched through that too, but I still did not find anything clearly pointing to a cave.

Since that was not working, I changed the approach.

I searched for caves near the after-party location and found a few possible options. One of the caves listed was:

```text
Gua Sumur
```

I tried that and it was correct.

This made sense with the scenario. The missing person probably did not randomly research a foreign cave by himself. Since he had met a local DJ, the DJ was probably the one who knew about the cave and suggested it as a local experience.

---

## Finding the DJ’s Tour Business Number

The final question asked:

```text
What number did the DJ list for his tour business?
Format: Full number, no country code.
```

At first, I went back through the original DJ Bong Facebook account and his Instagram again, but I still did not find the cave or tour number.

Then I searched Google for:

```text
Bong Leleh Gua Sumur
```

My thought process was that if Bong Leleh was connected to the cave, then searching his name together with the cave name should bring up the right business or tour page.

That search led me to a Facebook page connected to:

```text
@bongleleh
```

The page also referenced:

```text
Gua Sumur Lombok
```

I opened the Facebook page, went to the `About` tab, then checked the contact information.

There, I found the phone number.

The room asked for the full number with no country code, so I copied the number and submitted it.

It was wrong at first because the number had hyphens.

After removing the hyphens, the answer worked.

---

## Result

The investigation connected the clues in this order:

```text
Pertamina sign
Indonesia
Pertamina Mandalika International Street Circuit
2025 Indonesian MotoGP event
Cantina Mexicana
EXIF DateTimeOriginal timestamp
MotoGP after-party
Surfers Bar Kuta
Local Indonesian DJ
Bong Leleh
Gua Sumur
Tour business phone number
```

The room was not really about exploiting anything. It was more about slowing down, checking metadata properly, searching carefully, and not getting baited by the first result that looks correct.

---

## Takeaway

This lab was definitely different from the web vulnerability and terminal-heavy labs I had done before.

Most of the work was based on:

- analyzing images
- checking metadata
- reverse image searching
- reading captions carefully
- searching social media
- connecting locations, dates, and people together
- understanding when an online source is useful but not the final answer

The biggest mistake I made was focusing too much on the original article date from the MotoGP image. The article used the same image, but the room scenario clearly said the trip happened in 2025.

The biggest thing I learned was how EXIF metadata works and why the normal Windows properties view is not always enough. Windows showed the hour and minute, but the room needed the seconds too, which came from the raw `DateTimeOriginal` EXIF field.

Overall, this was a good OSINT lab. It felt more like solving a real-world internet mystery than running commands, but the pattern recognition was familiar because I have done a lot of image searching and internet digging before.
