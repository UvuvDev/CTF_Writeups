Description: the flag matches this regex: ^LITCTF\{[A-Z]+(?:_[A-Z]+)*\}$

This is a Unity Rev challenge. I did some research into typical CTF strategies and checked to see where people usually hide flags: Assembly-CSharp, and the resources. I downloaded ILSpy so I could decompile the Assembly-CSharp binary and see if there's any scripts in here. Looked around, nothing particularly interesting, though I did note that the function for completing a level has no bounds checking.

I looked into the resources of the game with UnityPy. The only real data was the level0,1,2...level7 files. My first instinct was that the flag might be in overlaying the different levels data with each other, but graphing it out gave me nothing.

<img width="806" height="582" alt="image" src="https://github.com/user-attachments/assets/52afcdcb-ca13-4ce1-82a7-72a18e92217e" />

I looked up if there was anywhere where scripts might be being called by the level, and it ended up being the MonoBehavior category, so I checked to see what they were doing. All of the scripts were in the original binary, so the flag isn't a custom script or verification check. First comes Menu, then 4 level completes, then the credits. BUT then there's 2 levels after!

<img width="520" height="582" alt="image" src="https://github.com/user-attachments/assets/1b12329b-63f2-405b-ab4b-87462152f17e" />

A secret level probably has the flag. I wanted to see what was in these levels, so I renamed level7 to level0 to see what happened. It caused this:

-- insert

There's an enormous amount of tiles, some of them are flying down from the horizon. I graphed the level data to see if there was anything interesting. It's really hard to see, but turn your head to left. Look around 250. LITCTF{ is visible. The flag is spelled out in the data.  
<img width="534" height="86" alt="image" src="https://github.com/user-attachments/assets/1eadc932-4d42-4618-915d-413b436f9a20" />  
I zoomed in, cut off everything past 250.  
<img width="1768" height="345" alt="image" src="https://github.com/user-attachments/assets/9cb36e8f-c3ac-4360-bdbb-7ced6bdd292a" />
Now we just draw over it.  Remember the Regex: the only allowed characters are uppercase and underscores!  
<img width="1759" height="401" alt="image" src="https://github.com/user-attachments/assets/9663ee20-cba5-47da-bf8b-9a5c906aae95" />

LITCTF{I_HAD_TOO_MUCH_FUN_MAKING_THIS}
