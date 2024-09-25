# Getting PaddleOCR and PaddlePaddle to work in Windows, Ubuntu and macOS

Working through the combinations to get PaddlePaddle and PaddleOCR installed and working everywhere

## Introduction

OCR is one of those standard AI vision-based features we're all familiar with. OCR is now so ubiquitous that it's built into cellphone operating systems and we barely notice. Obviously we needed to add it to [CodeProject.AI Server](https://www.codeproject.com/ai) but doing so was a bit of an adventure.

There are a ton of OCR projects and packages, and with Mike Lud leading the charge in picking the most accurate, we settled on [PaddleOCR](https://pypi.org/project/paddleocr/). PaddleOCR is based on the excellent [PaddlePaddle](https://pypi.org/project/paddlepaddle/) (PArallel Distributed Deep LEarning) package. Unfortunately PaddleOCR and PaddlePaddle can be a challenge to get working, so here's a quick rundown of what we did:

## Getting PaddlePaddle and PaddleOCR setup in Python

1. We ignored the docs. PaddleOCR evidently supports CUDA 10.6, but PaddlePaddle (which PaddleOCR needs) evidently only supports CUDA 10.1. Except where it states it supports 10.6. We have it running on CUDA 11.7. This confusion is pretty standard for evolving projects that are trying to target Nvidia. Anyone targeting Nvidia is brave, or they know how to lock their system to a specific set of drivers, toolkits and libraries. It's a mess.
2. We lived in Google Translate for 3 days. PaddlePaddle was developed by Baidu, the Chinese tech giant, and Open Sourced in 2016. Since it's based in China the forums are generally not English.
3. We experimented, read, tested, and adjusted and ended up with the following setup matrix for installing the Python packages:

 

    

| <!----> | <!----> | <!----> |
| --- | --- | --- |
    | OS | CPU | GPU |
    | Windows | paddlepaddle==2.3.2<br>
            paddleocr&gt;=2.0.1 | --find-links https://www.paddlepaddle.org.cn/whl/windows/mkl/avx/stable.html<br>
            paddlepaddle-gpu==2.3.2.post116<br>
            <br>
            paddleocr&gt;=2.0.1 |
    | Ubuntu | paddlepaddle==2.4.0rc0<br>
            paddleocr&gt;=2.6.0.1 | No success |
    | macOS (Intel) | paddlepaddle==2.3.2<br>
            paddleocr&gt;=2.0.1 | Not supported |
    | macOS (Arm64) | paddlepaddle==2.3.2<br>
            paddleocr&gt;=2.0.1 | Not supported |
4. Patched an ugly hack in the PaddlePaddle code to get things working.



In the `paddle` package under your site-packages folder in your Python installation (or virtual environment) you'll find a folder *dataset*, and within that the file *image.py*. Line #37 has a FIXME for the ugly hack to fix an issue with numpy when importing OpenCV. They import OpenCV by spinning up a new Python interpreter and import OpenCV directly:


    ```
    
        interpreter = sys.executable
        # Note(zhouwei): if use Python/C 'PyRun_SimpleString', 'sys.executable'
        # will be the C++ execubable on Windows
        if sys.platform == 'win32' and 'python.exe' not in interpreter:
            interpreter = sys.exec_prefix + os.sep + 'python.exe'
        import_cv2_proc = subprocess.Popen(
            [interpreter, "-c", "import cv2"],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            shell=True)
        out, err = import_cv2_proc.communicate()
        retcode = import_cv2_proc.poll()
        if retcode != 0:
            cv2 = None
        else:
            import cv2
    else:
        try:
            import cv2
        except ImportEr:...
    ```



That's some creative problem solving.



The issue is the process is spun up using `Popen` with `Shell=False` (specifically, they just let `Shell `take the default value, which is `False`). In the code sample above, at line 52, you can see the fix. For Windows and Ubuntu you need to have `Shell=True`, otherwise the import fails. For macOS `Shell=False `is fine (both Intel and Arm64).



Evidently this hack was only for Ubuntu, but it's in the Paddle code for all operating systems, so it needs to be fixed for both Windows and Ubuntu.

And with that we have PaddleOCR running. It's not very fast on a CPU. 10-15 seconds for half a page of text, but turn on GPU and it's 200ms or so. Totally useable and very accurate.

The next release of CodeProject.AI Server will include an option to install OCR using PaddleOCR. Our project is for the first week of December.

## Postscript: GPU support for PaddlePaddle in Ubuntu under WSL

This one has defeated us so far. PaddleOCR (CPU only) in Ubunut 22.04 on WSL works fine. GPU enabled does not. One major issue with WSL is you need to install CUDA in WSL rather than rely on the Windows CUDA drivers doing their thing. A write-up can be found [here](https://chennima.github.io/cuda-gpu-setup-for-paddle-on-windows-wsl). Once you have CUDA installed, test it be opening a Python terminal and entering

```cpp
import paddle
paddle.utils.run_check()
```

If this is all good then you can look at installing the Paddle packages, our best effort so far is using

```cpp
-f https://paddlepaddle.org.cn/whl/stable/noavx.html
paddlepaddle-gpu==2.4.0rc0
paddleocr==2.6.0.1
```

This installs succesfully and will at least launch (other combinations crash half a dozen different ways) but you end up with the very issue that the hack (above) was meant to solve: namely, you will get a segfault when trying to run PaddleOCR due to a numpy bug caused by importing OPenCV. The hack is meant to solve this by importing OpenCV in a spawned Python process. This process isn't successful in ubuntu 22.04 (at least for us) so it falls back to a classic "import cv2" which then leads to the segfault.
