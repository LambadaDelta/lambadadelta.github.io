## Background

Many are aware of the reduction being applied to Microsoft OneDrive from 15GB to 5GB, for me this wasnt too bad, I was only around 0.5 over.

However my wife who is the primary chronicler for the family was running around 10GB.

Either way there was a need to remove some content from OneDrive.

Some would say the easier option is to start using Google Drive, but where is the fun in that!

## Tools used

To solve this problem, I was planning on writing a PowerShell script which I was writing with PowerShell ISE. I also needed to check the files moving had worked and reduced the storage enough, for that I was using [TreeSize Free][treesize], and as the name suggests yes it is free.

## Looking at the Api

I started off looking at the api provided for onedrive [a demo of this is available on github for you to play with](https://github.com/OneDrive/onedrive-sdk-csharp), and there is a lot of fun that can be had with syncing a location on your machine with your onedrive.

However, I was constantly finding that I had to throttle the process a lot and if I was asking for too much too fast it would error out of the requests, waiting for even 5GB took some time.

I was also needing to manually authenticate against 2 different accounts to get all the files, would have also added some extra fun to the process. Let's be honest there is probably something in there that might mean it is possible to automatically flip between different accounts, but during my googling it hit me, I am already syncing a local copy of my OneDrive to my PC, if could share the camera folder from my Wife's OneDrive and add that to my local sync, I could just run a local script against the storage, and any changes (files I moved) would then be reflected in the cloud version.

I found an article that explained how to do it, and it was fairly straight forward.

## Sorting out the Sync (not the kitchen one)

Starting in the browser, I signed into my wife's account and checked that she was sharing her camera sync with me and I had edit access, she was (step 1 complete, big tick). This had pictures for the last few years and a few videos, as I said earlier, in total was around 10GB hosted in OneDrive. To Share is easy, to the top right of a directory is a little circle, clicking on this selects it, so you can do multiple in one go.

once you have selected all the files you are after you can click "Share" or you will also see something change on the right of thes screen, this tells you about the current level of sharing going on

![alt text][share]

From my OneDrive account (still in the browser), I clicked on shared and could see her camera drive shared with "Can Edit" showing.

![alt text][canedit]

I clicked the little circle that appears in the top right of the folder and then chose "Add to my OneDrive", that added the file to my local sync.

![alt text][addtomyonedrive]

I didnt have a folder called Camera Roll, not sure what would have happened if I did.

## Next steps

I had to do was wait for everything to sync up, we were talking about 10GB here so that took a while. Once that was done I ran TreeSize Free against my OneDrive directory and confirmed that the full camera was included. Then I ran a script to remove the videos, I thought that might have been enough, so did the following.

```powershell
$drive = "D:\remoteDrives\MyOneDrive"
$stage = "D:\remoteDrives\Stage"

#Do the videos first
$items = Get-ChildItem -Path $drive -Recurse -Include *.mp4, *.mov

$items.ForEach({
$item = $_
$to = $item.FullName.Replace($drive,$stage)
$toPath = Split-Path -Path $to -Parent
$toPath = "$toPath\"
"$($item.FullName)---$($toPath)"
if(!(Test-Path $toPath))
{
New-Item $toPath -Type Directory
}

Move-Item -Path $item.FullName -Destination $to
})
```

_Just as an aside_ an interesting shortcut to remember in the ISE is `Ctrl + D`, this takes you to the command line, shown to be by a wise PowerShell Yogi / mentor and friend.

## Say what I see

The behaviour is quite straight forward, the drive is mapped to where the files are and the stage is mapped to where I want them to go.

What I then do with them is another story, I just wanted to get them primarily off the OneDrive and have some control about it.

So, I build a list of videos of mp4 and mov format in the location of my drive. Then for each of the items I replace the drive location with the stage location on the path, generate the directory structure and then move the file.

_Now another aside_, there is a difference between `Copy-Item` and `Move-Item`, `Copy-Item` will give you an option of `-Recursive`, which means if the directory that the file lives in does not exist, it will created it as part of the process, `Move-Item` doesnt have this luxury.

After I had moved the videos I checked, it had helped, but I was still a few GB over the 5GB limit, rather than worry about size of things I thought I would then concentrate on date, as these are picture images, there is a property called `LastModifiedDate`, this is usually when the picture was taken, if you modified your picture, that would be set here when you did that.

So I chose an arbitrary buffer of 6 months (and ideally, anything older would be moved). I picked all the file types I was interested in.

Appologies in advance for the duplication you may see, I did this for ease of reading the process and also at the time I wasnt sure if I could do the same.

```powershell
#now pictures, but only if they are older than 6 months
$g = Get-Date
$a = $g.AddMonths(-6)

$items = Get-ChildItem -Path $drive -Recurse -Include ("*.jpg", "*.png", "*.jpeg") |
Where-Object {$_.LastWriteTime.ToUniversalTime() -lt $a.ToUniversalTime() `
-and $_.FullName -notlike "*Wedding*" `
-and $_.FullName -notlike "*Dogs*" `
-and $_.FullName -notlike "*CompanySecret*" }

$items.ForEach({
$item = $_
$to = $item.FullName.Replace($drive,$stage)
$toPath = Split-Path -Path $to -Parent
$toPath = "$toPath\"
"$($item.FullName)---$($toPath)"
if(!(Test-Path $toPath))
{
New-Item $toPath -Type Directory
}

Move-Item -Path $item.FullName -Destination $to
})
```

I only wanted known file types, but also to leave collections that I knew existed , until later (possibly), those are defined in the `Where-Object`, comparing the `FullName` to some values I knew existed in the strings.

You may see the \` at the end of some lines, this allows your code to extend to a new line.

After all that was done, I was able to reduce my wife's total size down to around 3GB, and I regularly run the move script to keep it under control. There is still some refactoring work and turning this into a `cmdlet` and scheduling the task, should help reduce the work needed by me each time.

## Is there an easier option / alternatives?

For free? Yes! Move to Google Drive, but as I said at the start, where is the journey and the fun in developing something to solve this first world problem!

They still have (at the time of writing this post) 15GB for free, they even offer a process that if you allow them to convert the picture to a high quality version you wont use any of your quota, it probably does degrade the quality of the picture, but unless you are expecting to do something fancy I dont think you would always need to hold on to the full image generated anyway.

Paid, Microsoft are offering the 15GB limit if you buy an Office 365 licence (for around £70 a year).

Microsoft also ofter a friends scheme, if you introduce people to one drive you get a boost.

[canedit]: canedit.png "Image of Camera Roll directory with Can Edit and checked"
[addtomyonedrive]: addtomyonedrive.png "Add To OneDrive link"
[share]: share.png "Share OneDrive directory"

[treesize]: https://www.jam-software.com/treesize_free/