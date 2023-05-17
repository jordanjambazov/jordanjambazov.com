---
layout: post
title: "Bisecting vandalism"
date: 2023-05-17 09:00:00 +0200
categories: misc
---

Not too long ago - something exciting happened. The municipality built a new playground not too far from my home. As a father of 2 kids below 3, I was more than happy with this. Soon my joy was superseded by a feeling of frustration. Day after day, boards from the playground fence started to break and disappear. The playground was a victim of vandals.

![Vandalism on the playground](/assets/images/vandalism-playground.jpeg)

What to do about it? I decided to install a surveillance camera and start tracking the acts of vandalism. The people from the house next to the playground provided the infrastructure - internet and electricity. A friend of mine helped with the camera installation. We set up the camera to record whenever there is motion and upload the videos to an FTP server hosted within my home lab. Soon there were hundreds of short video clips. A month later, the next set of broken fence boards appeared.

Of course, it would have been suboptimal to watch thousands of videos. I binary searched the records using an approach similar to Git bisect. I extracted a list of all the files on the FTP server and sorted it. By specifying "good" and "bad" files within the range, it is possible to find the first "bad" occurrence with logarithmic complexity. A "good" file is when the vandalism did not happen, and a "bad" file is when the marks are visible.

The backbone of the bisect implementation is this Python function:

```python
def bisect_files(files, good, bad, pre_hook):
    if good not in files or bad not in files:
        raise ValueError("Good and bad files must be in the list.")
    
    if files.index(good) > files.index(bad):
        raise ValueError("Bad file must come after good file in the list.")
        
    left = files.index(good)
    right = files.index(bad)

    while right - left > 1:
        mid = (left + right) // 2
        file_name = files[mid]
        pre_hook(file_name)
        print(f"Is file '{file_name}' good or bad?")
        response = input().lower().strip()
        
        if response not in ['good', 'bad']:
            print("Invalid response. Please respond with 'good' or 'bad'.")
            continue
        
        if response == 'good':
            left = mid
        else:
            right = mid

    print(f"The first bad file is '{files[right]}'")
```

The `pre_hook` is a custom callback function that downloads the video on each bisect iteration and opens it.

```python
def pre_hook(file_name):
    ftp = FTP('192.168.1.146')
    ftp.login(user="playground")
    try:
        print(f"Starting download of remote file: {file_name}")
        local_file_name = file_name.replace('/', '_')
        with open(local_file_name, 'wb') as fh:
            ftp.retrbinary('RETR ' + file_name, fh.write)
        print(f"Downloaded file {local_file_name}")
        os.system(f'open "{local_file_name}"')
    finally:
        ftp.quit()
```


Using this approach, I successfully identified 3 acts of vandalism. All of them happened to be kids. We are currently working with their parents to recover the playground.
