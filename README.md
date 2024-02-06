# Masive download for car musik mp3 playlist

Install all the tools:
```
sudo apt install parallel detox normalize-audio mp3gain
wget https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -O ~/.local/bin/yt-dlp
chmod a+rx ~/.local/bin/yt-dlp  
yt-dlp --update-to nightly
```

Use a cookie in your home directory. There is a Firefox extention to save a cookie in a file: https://github.com/hrdl-github/cookies-txt. Save it as ~/cookies.txt

Open your web browser and navigate to YouTube. Use the search function to look for playlists and music genres. For example, search for 'salsa 2023 playlist'. Press the playlist button right below the search field. Go as much page down as you like. If you're happy, save the open tag as a text file. Scan the HTML file with the next command of the saved playlist to retrieve the tags.
```
cat ~/Downloads/salsa\ 2023\ playlist\ -\ YouTube.html | grep "playlist?list=PL" | \
sed -n 's/.*list=\(.*\)>.*/\1/p'  | tr '\n' ' '
```
You should get something like this:

PLD0kvNhPZ444CoLU7Z2ri3nbMn6uVDscR PLXl9q53Jut6k_WLWfIK3zv-3kwnBnA5fm PLJzWprC5a8Ad49KnLX6_FgX0VAsp8J-h1 PLUMJYOoO2JQ_DcCSmuFiKRTk6J5vJcrlH PLgFPSBWI2ATu3JE4tCZqKaaXhNBex-t7o PL8rVmOSQfvq8lLXTfu5UzQPilRwB_xzoN PL4U35lg0iKyZGrx9YITNqfgBwlah7Rm8A PLgvKCwa4Uw1t8s8vfi5VuOisQJNsuoXkX PLxf7wRx2pn4t-TYq6tAKPnosiPRjShmAy 

But you can also hand select the playlists and collect the tags to get a list like this. For a car playlist, maybe 1000 songs is sufficient. So do not collect too many playlists as it will produce too many songs.

I selected here probably something like 1098 mp3 Songs. You can check it with the command below:
```
time nice yt-dlp --print "https://www.youtube.com/watch?v=%(id)s" --flat-playlist PLD0kvNhPZ444CoLU7Z2ri3nbMn6uVDscR PLXl9q53Jut6k_WLWfIK3zv-3kwnBnA5fm PLJzWprC5a8Ad49KnLX6_FgX0VAsp8J-h1 PLUMJYOoO2JQ_DcCSmuFiKRTk6J5vJcrlH PLgFPSBWI2ATu3JE4tCZqKaaXhNBex-t7o PL8rVmOSQfvq8lLXTfu5UzQPilRwB_xzoN PL4U35lg0iKyZGrx9YITNqfgBwlah7Rm8A PLgvKCwa4Uw1t8s8vfi5VuOisQJNsuoXkX PLxf7wRx2pn4t-TYq6tAKPnosiPRjShmAy | wc -l
```
Now it's time to massive download (this takes 15min):
```
time nice yt-dlp --print "https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(title)s.mp3" --flat-playlist PLD0kvNhPZ444CoLU7Z2ri3nbMn6uVDscR PLXl9q53Jut6k_WLWfIK3zv-3kwnBnA5fm PLJzWprC5a8Ad49KnLX6_FgX0VAsp8J-h1 PLUMJYOoO2JQ_DcCSmuFiKRTk6J5vJcrlH PLgFPSBWI2ATu3JE4tCZqKaaXhNBex-t7o PL8rVmOSQfvq8lLXTfu5UzQPilRwB_xzoN PL4U35lg0iKyZGrx9YITNqfgBwlah7Rm8A PLgvKCwa4Uw1t8s8vfi5VuOisQJNsuoXkX PLxf7wRx2pn4t-TYq6tAKPnosiPRjShmAy | awk -F';' '{print "yt-dlp --ignore-errors -no-abort-on-error --no-check-certificate --cookies ~/cookies.txt --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata " $1 " -o \"~/Downloads/SalasPlaylist/" $2 "/" $3"\""}' | nice ionice -c 3 parallel --ungroup --eta -P20
```
Clean up filenames with detox - the car stereo likes clean filenames:
```
ionice -c 3 detox -vr ~/Downloads/SalasPlaylist
```
Delete too big and too small files. Check first with "echo" in front of "rm" and maybe "| wc -l" at the end:
```
find ~/Downloads/SalasPlaylist -type f -name "*.mp3" \( -size -3M -o -size +8M \) -exec rm {} \; 
```
Delete files and subdirectories that are more than 2 levels deep from ~/Downloads/SalasPlaylist. Check first with "echo" in front of "rm" and maybe "| wc -l" at the end:
```
find ~/Downloads/SalasPlaylist -mindepth 2 -type d  -exec rm -rf {} \;
```
Normalize, with audio file volume normalizer. Depends on your audio player to work. Maybe there is a better way.
```
time find ~/Downloads/SalasPlaylist -type f -name "*.mp3" | parallel --eta -P20 nice ionice -c 3 normalize-audio {}
time find ~/Downloads/SalasPlaylist -type f -name "*.mp3" | parallel --eta -P20 nice ionice -c 3 mp3gain {}
```

Next on the to-do list: 
- Remove all folders containing music you do not like.
- Shorten directory names.
- Copy to an MP3 stick.
- Erase from the hard disk.

Have fun and enjoy the ride :=)
