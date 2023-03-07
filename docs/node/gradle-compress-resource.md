```

project.afterEvaluate {
    android.applicationVariants.all { variant ->
        if (variant.name.contains('debug')) {
            def compress = new ArrayList<String>()
            variant.allRawAndroidResources.files.collect {
                searchImage(it, compress)
            }
            compressImage(compress)
        }
    }
}

private void compressImage(ArrayList<String> files) {
    files.collect {
        def file = new File(it)
        if (isJpg(file)) {
            def oldSize = file.size() / 1024
            def result = executeCmd("$rootDir.path/imagetools/mac/guetzli $file.path $file.path")
            if (result) {
                def newSize = file.size() / 1024
                print("compress success: $file.path $oldSize -> $newSize")
            } else {
                print("compress fail:$file.path")
            }
        }
    }
}

private void searchImage(File file, ArrayList<String> directory) {
    if (file.isDirectory()) {
        file.listFiles()?.collect {
            if (it.isDirectory()) {
                searchImage(it, directory)
            } else if ((isPng(it) || isJpg(it) || isWebp(it)) && !it.name.endsWith(".9.png")) {
                if (it.size() > 200 * 1024) {
                    directory.add(it.path)
                }
            }
        }
    }
}

private boolean isPng(File it) {
    return it.name.endsWith('.png')
}

private boolean isWebp(File it) {
    return it.name.endsWith('.webp')
}

private boolean isJpg(File it) {
    return it.name.endsWith('.jpg') || it.name.endsWith('.jpeg')
}

private boolean executeCmd(String cmd) {
    try {
        def process = Runtime.getRuntime().exec(cmd)
        process.waitFor()
    } catch (Exception e) {
        print(e.message)
        return false
    }
    return true
}
```