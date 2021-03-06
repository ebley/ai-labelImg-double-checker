# ai-labelImg-double-checker
Unless you are perfect - having some way to double check your classifications from LabelImg is a must. I didn't find anything so I made one - it is super simple. It extracts all images with the file name .check.1.MyClass1Name at the end. Then you can look at them all side by side and see how you did with a simple Explorer or Finder window filtering each class one at a time. I am sadly not perfect I found out.

Here is what it does
a) for every .png/.jpg with a .txt that isn't empty it creates a sub-image clip just like you marked in LabelImg (or the like) in YOLO format (other format to come) 
b) It tallies all the samples for each class
c) it makes a full-path list of images that were actually processed (leaving the empty .txt file items OUT of the final list)
    - this is used later on in the YOLO setup anyway - and I am NOT going to type them all out by hand :-)
d) If something goes wrong it tells you along the way. If all goes well there is a % progress is all. 
e) It prints a summary at the bottom
    - of how many images per class
    - where the output directory is in case you forget or are lazy like me

Right now (cause I really got to crack on!) it only does YOLOv2/3 format - I am sure I will modify it (or someone else can) for Pascal VOC format. With YOLO, for heaven's sake remember that the x,y is the center of the square.

# install
Well it is one file so just ensure you have opencv (a reminder that you cannot just `pip install cv2` - you have to type `pip install opencv-python` and then `import cv2`

# instructions
So if we make an image extractor - we can quickly check that our LabelImg efforts worked.

Set the sourcedir variable to the proper directory. change `diffCheckDirExt` if you want photos saved in another peer directory to your images directory.

It looks for jpg and pngs and if there is a corresponding .txt file it iterates through every line and makes a file called .check.X.<yourclassname\>.jpg (or .png). X could be the class number starting from 0 in a file called classes.txt that needs to be along side your images - its the same class file you would use for YOLO or the like.

You have to have a classes.txt file that would have come from your LabelImg efforts anyway. I assume it is in the same directory as the images - which might not be the case for training - but I wanted to keep this simple.

# code
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
        if item == "file_list.txt":
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
        if index == 0:
            file_object = open(os.path.join(sourcedir+diffCheckDirExt,'file_list.txt'), 'a')
            halfpath = filename
            file_object.write(halfpath + "\n")
            file_object.close()

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
    print("")
    printSummary()

def printSummary():
    print("Counts")
    print("---------------------------------------")
    for i in range(len(classDef)):
        print ("Class " + str(i) + ": " + classDef[i] + ":" + str(classDefCount[i]))
    print("")
    print ("Total Successful/Valid Extractions: " + str( round(totalValidExtractions/totalExtractions*100) )+ "%")
    print ("A reminder that your check folder is: " + sourcedir + diffCheckDirExt)
    print ("There is also a full-path files_list.txt file there ")
    
mainRoutine()

```
