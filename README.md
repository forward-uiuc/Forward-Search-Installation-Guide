# Forward-Search-Installation-Guide
This contains step-by-step guide to set up Forward Search for a particular domain. Input is a URL template for the target domain and output is a web-based entity-semantic search engine for the domain.

# Crawl Data
You need to install Jupyter Notebook to open file `PrepareUrlList.ipynb` in https://github.com/forward-uiuc/Common-Crawl-URL-Searcher. Change the domain in cell 6, e.g., `domain = "forwarddatalab.org"`, and run all cells. If necessary, please install missing packages by using pip, such as `pip install url_normalize`. Then you will find a new CSV file containing all urls corresponding to the domain in the same folder as the notebook file.

# Annotate Data
Clone repository https://github.com/forward-uiuc/Entity-Search-Annotation-Indexing. 

Go to the directory of the code repository you just cloned. Run `mvn package` to compile and create jar file. The repository is tested with Java 8.

Copy the csv created in the step above to folder input in the directory.
Go to this link http://chromedriver.chromium.org/downloads to download the chromedriver compatible with your operating system. Copy the driver file to lib folder in the directory. 

Open `run.sh` to understand what it does. Open terminal and run `sh run.sh $domain $output`, in which `$domain.csv` is the csv file you in the step created above, and `$output` is the output folder of the serialized file. For example, `./run.sh forwarddatalab.org ../ouptut/`.  You should create the output folder `output` ahead. The syntax is fragile so please follow the provided example precisely.

You can check log file in `output/$domain/log` to see errors. You can also run `ps -ef | grep chrome` to see if chrome driver is running. Please note that this code is for production so I set the flag to not show Chrome UI. So you may see it is not running but it is actually running. Check the log files to see updates. For each done URL, an entry is noted on std file in the log folder. For each URL, this step downloads the file, and does expensive annotation, so it is relatively expensive for personal computer. You may expect 1-2 minutes for an URL. For a big domain, it could take days to finish.
