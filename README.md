# ai-labelImg-double-checker
Unless you are perfect - having some way to double check your classifications from LabelImg is a must. I didn't find anything so I made one - it is super simple. It extracts all images with the file name .check.1.MyClass1Name at the end. Then you can look at them all side by side and see how you did with a simple Explorer or Finder window filtering each class one at a time. I am sadly not perfect I found out.

Oh, and only outputs errors, it puts a summary of how many of each class it sucesssfully saved.

By default - if you directory you picked is img then it saves it in a new peer folder called img.check

#install
Well it is one file so just ensure you have opencv (a reminder that you cannot just `pip install cv2` - you have to type `pip install opencv-python` and then `import cv2`

#Code to double check that you did the YOLO part correctly - look at your images
So if we make an image extractor - we can quickly check that our LabelImg efforts worked.

Set the sourcedir variable to the proper directory. change `diffCheckDirExt` if you want photos saved in another peer directory to your images directory.

It looks for jpg and pngs and if there is a corresponding .txt file it iterates through every line and makes a file called .check.<yourclassname\>.jpg (or .png)

You have to have a classes.txt file that would have come from your LabelImg efforts anyway. I assume it is in the same directory as the images - which might not be the case for training - but I wanted to keep this simple.

#code
``` python
import cv2
import numpy as np 
import pandas as pd 



sourcedir = "D:\\img"
diffCheckDirExt = ".check"
classDef = {}
classDefCount = {}
totalExtractions = 0
totalValidExtractions = 0 

def loadClasses():
    cf =  os.path.join(sourcedir,"classes.txt")
    #print(cf)
    if not os.path.exists(cf):  
        print("classes.txt file missing. Make one or add one" + cf)
        return False
    #print(cf)
    classes = pd.read_csv(cf, names = ['class'])
    print (classes)
    if not classes.count:
        return False
    
    i = 0

    for index, row in classes.iterrows(): 
        classDef[index] = row['class']
        classDefCount[index] = 0

    classes['countItems'] = 0
    #print (classDef)
    

    return True

def makeCheckDir():
    if not os.path.exists(sourcedir + diffCheckDirExt):
        os.mkdir(sourcedir + diffCheckDirExt)
    return

def preDeleteFilesOfType(intype):
    test = os.listdir(sourcedir + diffCheckDirExt)

    for item in test:
        if (item.endswith(".png") or item.endswith(".jpg")) and item.find(".check.")>=0:
            os.remove(os.path.join(sourcedir + diffCheckDirExt, item))

def handlefile(file):
    global totalExtractions
    global totalValidExtractions
    #check for .txt file with the same name
    filename = os.path.join(sourcedir,file)
    pre, ext = extension = os.path.splitext(file)
    txtFile = os.path.join(sourcedir,pre + ".txt")
    if not os.path.exists(txtFile):
        print ("Image.txt file does not exist"+txtFile)
        return
    
    tf = pd.read_csv(txtFile,delim_whitespace=True, names = ['class','xmid','ymid','w','h'])

    im = cv2.imread(filename)
    if im is None:
        print("Can't load image: " + filename)
        return
    
    for index, row in tf.iterrows():
        totalExtractions = totalExtractions  + 1
        if exportFile(index, row, pre,ext,im):
            classDefCount[int(row['class'])] = classDefCount[int(row['class'])] + 1 #the index is numeric version of the class
            totalValidExtractions = totalValidExtractions + 1


    
            

def exportFile(index, row , pre, ext, im):

    imh = im.shape[0]-1
    imw = im.shape[1]-1
    
    #print('Image',imw, imh, str(round(row['class'])))
    x1 = round(((row['xmid'] - row['w']/2))*imw)
    x2 = round(((row['xmid'] + row['w']/2))*imw)
    y1 = round(((row['ymid'] - row['h']/2))*imh)
    y2 = round(((row['ymid'] + row['h']/2))*imh)
    #print('Image',imw, imh, x1,y1,x2,y2)
    im2 = im[y1:y2,x1:x2]
    #if im2.empty():
    #    print("Image is empty")
    #    return 

    outfn = classDef[round(row['class'])]
    outFile = os.path.join(sourcedir + diffCheckDirExt,pre + ".check."+ str(index) + "." + outfn+ext)
    #print("Writing:"+outFile)

    sh2 = im2.shape
    if sh2[0] == 0 or sh2[1] == 0:
        print ("Can't write (img empty):" +outFile)
        return False

    if not cv2.imwrite(outFile, im2):
        print ("Can't write: " + outFile)
        return False
    return True

def mainRoutine():
    filename = "testing.jpg" # was used for testing

    if not loadClasses():
        sys.exit()
    makeCheckDir()
    preDeleteFilesOfType(['.png','.jpg'])

    #handlefile(filename)
    filess = os.listdir(sourcedir)
    filessCount = len(filess)
    print (str(filessCount) + " images")
    i=0.0001
    for file in filess:
        i = i + 1
        if file.endswith(".jpg") or file.endswith(".png"):
            percentComplete = i / filessCount * 100.0
            #print(os.path.join(sourcedir, file))
            print(str(round(percentComplete)) + '% ' + os.path.join(sourcedir, file), end='\r')
            handlefile(file)
    print ("100% Complete")
    printSummary()

def printSummary():
    print("Counts")
    print("---------------------------------------")
    for i in range(len(classDef)):
        print ("Class " + str(i) + ": " + classDef[i] + ":" + str(classDefCount[i]))
    print("")
    print ("Total Successful/Valid Extractions: " + str( round(totalValidExtractions/totalExtractions*100) )+ "%")
    print ("A reminder that your check folder is: " + sourcedir + diffCheckDirExt)
        
mainRoutine()

```
