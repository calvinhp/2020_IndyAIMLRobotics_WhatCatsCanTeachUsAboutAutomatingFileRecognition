---
title: What Cats Can Teach Us About Automating File Recognition
author: Calvin Hendryx-Parker, CTO, Six Feet Up
date: Indy AI ML Robotics Feb 2020
---

# About me {.semi-filtered data-background-image="images/AWSHeros.jpeg"}

# Challenge {data-background-image="images/14555354976_432207b15b_k.jpg"}
### 3 Key Challenges

- Human Factor = Bottleneck
- Human Review process = Errors
- “Lottery” factor / “Silver Tsunami” Problem

<small>Photo Credit <https://flic.kr/p/obcZW9></small>

# Opportunity {.semi-filtered data-background-image="images/cat-694730_960_720.jpg"}
### Process Bundles of Images in PDF Files

::: notes
- Leverage automation technologies to identify and process pages within PDF bundles
- Current workflow, here are some photos
:::

# Obstacles {.semi-filtered data-background-image="images/2961343284_699802a5cf_b.jpg"}
## Poor quality

- Illegible and poorly scanned
- Warped
- Blurry
- Faint or lacking contrast
- Handwritten

<small>Photo Credit <https://flic.kr/p/5vFEbQ></small>

# Obstacles {data-background-image="images/20728747542_1bb67d91af_h.jpg"}
## Lack of Consistency
- Sideways
- Upside down
- Out of order
- Missing

<small>Photo Credit <https://flic.kr/p/xzJft1></small>

::: notes
Documents are uploaded on the road through a phone camera
:::

# Advantages {data-background-image="images/24425384322_c3448900b1_h.jpg"}
- We know almost all document types we expect to receive
- Most of the documents are using similar shapes/patterns and orders
- Many documents contain machine readable identifiers - barcodes/QR codes


# Initial Exploration
### Google Cloud Vision (GCV)

![](images/1*eIMgK4vNuB5vWsB3EQIYIA.png)

# {data-background-image="images/1*eIMgK4vNuB5vWsB3EQIYIA.png"}

- Can integrate with existing software stack. 
- Has better than average ability to process documents for Optical Character Recognition (OCR)
- Allowed us to consistently identify poor quality documents by returning keywords


# {data-background-image="images/2u3s33.jpg"}
### QR Processing

- When processed successfully, document type identification returned 100%
- Less reliable when document quality was poor

::: notes
GCV was an attempt to just use OCR, GVC is page based, you can’t chop up the page to get useful context.

GCV couldn’t process the QR codes at the same time. It is pretty much a couple trick pony.

The QR codes we had available were too small, had too much data packed in them, became very dense and were vulnerable to compression on the phones that were scanning the documents.

We believed that understanding and breaking down the documents into machine readable components would lead to successful document identification. We researched various cloud automation technology for comparison.

Used GCV and OpenCV for QR code processing, still using OpenCV later in the workflow.

QR codes can’t be relied on due to the low resolution of the pages.

Data used in the QR code is to give us greater confidence

Once we know what form we have, we could cut up the form and OCR individual sections on their own.

OCR’ing the whole doc ends up with just a word jumble.

Classification against the words is difficult, you have to throw out must words such as stop words and there are words on the doc that are used in specific contexts such as font size etc.
:::

# Exploration in Machine Learning (ML)


# PyTorch, Keras and Tensorflow
- You have to do more
- You have to set it up and tune it
- You have to host it and manage the infrastructure

# Google AutoML
- Easy Setup
- Train the models
- 95%+ accuracy
- It’s pretty awesome

::: notes

- With a well-maintained set of training data (representative sample of at least 8 document types), AutoML is capable of successfully identifying and categorizing with 95%+ accuracy
- Using AutoML is much more like PyTorch, you don’t have to specify up front the number of layers upfront. It will break it up into 8 nodes to start and goes to town. You can pay extra to go wider.
- You can do 1 node or 8 nodes due to constraints made by Google
- We recommend you just click the parallel option and go fast

:::

# Workflow

---

![](images/automl_dashboard.png){.stretch}

---

![](images/automl_import.png){.stretch}

---

![](images/automl_datasets.png){.stretch}

---

![](images/automl_images.png){.stretch}

---

![](images/automl_eval.png){.stretch}

---

![](images/automl_test.png){.stretch}

---

![](images/automl_deploy.png){.stretch}

---

#### Flow for learning/training

encrypted pdfs dropped on Cloud Storage → Cloud Function to decode, deconstruct → drops page manifest → Cloud Function to pick up the pages → first pass at classification using perceptual hashing → cataloged by initial groups which are checked/edited by a “human in the loop” → create train/test/verify sets → passed into AutoML to create the model → model is then deployed → images removed from GS, but metadata kept as a JSON in GS.

::: notes

- Breaking documents down into their base components - vertical line processing
- By thinking of documents as repeating patterns or wavelengths, we can distill documents into groups of like patterns
- Using this method of wavelength processing, we were successful in pre-processing documents into similar groups to:
    - inform our training sets
    - identify groups of document types that may not yet be identified
        - successfully solving the “other” doc type challenge
:::

---

#### Flow for inferring

photos in PDF → deconstruct into pages → upload to Cloud Storage → JSON manifest is generated → manifest sent to a Cloud Function → Cloud Function picks up and deconstructs the pdf, writes a page level manfiest → Cloud Function picks up the images, passes them through pipeline including the deployed model → writes back final manifest with the results → manifest passed back to the client by webhook.

::: notes
Cloud Storage Events based on the JSON files uploaded.

Perceptual Hashing, Attempting to get similar checksum for the documents to get them to group together. Do they look similar? for example, the horizontal bars of the form aren’t  helpful, but the vertical bars are.

What are we encrypting and where? 
Files needs to be encrypted at rest all of the time, currently using google storage built in encryption.

How do we manage the secrets/keys?

The way forward may be Vault, but using unique keys for each file may be unrealistic.
:::

# Downsides of AutoML
- Potentially costly
  - <https://cloud.google.com/vision/automl/pricing>
- Endless maintenance required
- Deployment of the model takes an hour
- Spinning it down take 30-60 minutes
- Locked in to AutoML

::: notes
Single model runs at a minimum of $30/day if you don’t do any inferrance. Usage is pretty cheap.

$3.15/node/hour during training  (we used 8 nodes to get it down to 1 hour)
$1.25/node/hour to exist
1M predictions takes the costs to $40/day so just $10/day extra for 1M images.


- Getting to a 96% confidence rate, it took around 8 hours, but when spread across 8 nodes it took less than an hour.
- While potentially costly to maintain in an **always ready state**, the results of AutoML were impressive
- ML training sets must be maintained and evolve with an ever changing set of document types (**What if we have a document type come through that is not yet identified - the “Others” category**)
- Running inference is pricy, cost per classification is cheap, but keeping the model online is expensive. Significant ramp up time for inference (40-60 minutes to load) Around $30/day
- With AutoML, can you download the model?  We don’t think so
- Inference requires GPU instances that are in Iowa <insert caucus joke here>
- Very black box, we aren’t sure why it is slow to load and unload

In the beginning, the re-training should happen a lot.
We can get the model to tell us they don’t know what something is so a human can classify
first pass is happening with Python using numpy, pandas and Scipy, this is just hashing, no AI yet
first pass can run on thousands of images in minutes
This is how we get a high quality training set
Tools for grouping a head help mitigate how much training we may need to do
:::

# Quick Tip

- Use Click to Parameratize everything to make quick tools to run
- Not using Jupyter notebooks anymore

::: notes
due to cells state being able to execute out of order, makes for lots of noise and can fool you easily into thinking something is working.
:::

# Takeaways

# Just Do It!

![](https://imgs.xkcd.com/comics/machine_learning.png)

<https://imgs.xkcd.com/comics/machine_learning.png>

::: notes
You don’t know how any particular algorithm will perform until you try it

Start with AutoML, it is low friction.

If AutoML works well, stick with it

If it doesn’t, you have lots of options, move to PyTorch etc. But be ready for a lot of work.
:::

# GCV for OCR

![Image result for cat reading](https://images-na.ssl-images-amazon.com/images/I/71a5UsLFdrL._AC_SX466_.jpg)


::: notes
- Google Cloud Vision appears to be one of the top technologies available for OCR, based on our quality comparisons
AWS Tools weren’t as accurate and would need more cleaning pre and post
GCV OCR just seemed to nail it
:::

# AutoML for Patterns

::: notes
- Google AutoML is a viable candidate for pattern recognition for traditional paper documents, not just photography 
- Google AutoML takes a significant amount of time and system resources to spin up, but does return impressive results
::: 

# Don’t Put All Your Data in the same ML Basket
- Pattern recognition of Google AutoML trumps OCR for document identification, however relying on a single technology alone isn’t a silver bullet

# Know your data
- This is all about you

# Ask Me

## <calvin@sixfeetup.com>

### @calvinhp
