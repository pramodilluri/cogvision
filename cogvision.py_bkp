#!/usr/local/bin/python3.9


import flask
from flask import request, jsonify
import json

app = flask.Flask(__name__)
app.config["DEBUG"] = True

import argparse
import dateutil
from datetime import datetime 
from dateutil import parser as date_parser
from dateutil.relativedelta import relativedelta
import re

@app.route('/', methods=['GET'])
def home():
    return "Testing cogvision service"

@app.route('/productexpiry', methods=['POST'])
def expiry_handler():

    global mfg_flag, exp_flag, mfg_date, exp_date
    mfg_flag = False
    exp_flag = False
    mfg_date = ""
    exp_date = ""
 
    print(request) 
    req = request.get_json()
    print(req)
    path = req.get('path')
    url = req.get('url')
    
    if path:
        text = detect_text(path)
    
    if url:
        text = detect_text_uri(url)
    
    resp = is_expired(text)
    return jsonify({'response': { 'mfg-date': mfg_date, 'expiry-date': exp_date, 'expired' : resp}})


@app.route('/productharm', methods=['POST'])
def harm_handler():

    global harm_elements
    harm_elements = []

    req = request.get_json()
    print(req)

    path = req.get('path')
    url = req.get('url')
    
    if path:
        text = detect_text(path)
    
    if url:
        text = detect_text_uri(url)
    
    resp = is_harmful(text)
    return jsonify({'response': {'IS_HARMFUL' : resp, 'harmful_elements' : harm_elements}})


mfg_keys = ['mfg', 'manufactured', 'mfd']
mfg_not_keys = ['by', 'lic', 'license', 'bn', 'batch', 'bnum']
exp_keys = ['exp', 'use before', 'best before']


# keep adding based on learning
harm_list = [
		"sodium lauryl sulphate",
		"sodium aureth sulphate",
		"dimethicone",
		"parabens",
                "laureth"
	    ]

harm_elements = []

# Retail CRMs have product catelogue for which shelf life of products is always constant
# default 12 months for now
default_exp = 12

mfg_flag = False
exp_flag = False

mfg_date = ""
exp_date = ""

def is_date(string, fuzzy=False):
    #Return whether the string can be interpreted as a date.
    try:
        date_parser.parse(string, fuzzy=fuzzy)
        
        list_not = mfg_not_keys
        for key in list_not:
            if string.find(key) != -1:
                return False
        return True

    except:
        return False

def key_exists(line = "", flag = ""):
    ret = False

    if flag == "mfg":
        list = mfg_keys
        list_not = mfg_not_keys
    elif flag == "exp":
        list = exp_keys
        list_not = []
                
    for key in list:
        if line.find(key) != -1 :
            ret = True

    for key in list_not:
        if line.find(key) != -1:
            ret = False

    return ret



def is_expired(content):

    global mfg_date, exp_date
    global mfg_flag, exp_flag
    content = content.split('\n')
    
    for line in content:

        print(line)
        line = line.lower()

        if not mfg_flag and key_exists(line, "mfg") :
            mfg_flag = True
            print ("mfg identified")

        if not exp_flag and key_exists(line, "exp") :
            exp_flag = True
            print ("exp identified")

        if (is_date(line, True)): 
            date_text = date_parser.parse(line, fuzzy=True)
            if date_text:
                if key_exists(line, "mfg"):
                    #if not mfg_date:
                        mfg_date = date_text
                        print("mfg date identified %s" % (mfg_date))
                        continue 
                elif key_exists(line, "exp"):
                    #if not exp_date:
                        exp_date = date_text
                        print("exp date identified %s" % (exp_date))
                        continue 

                # no match and just date - Assume in the order of mfg and exp if key is already seen.
                if not mfg_date:
                    mfg_date = date_text
                    print("mfg date set %s" % (mfg_date))
                elif not exp_date:
                    exp_date = date_text
                    # swap if some products list exp date ahead of mfg date
                    if exp_date < mfg_date:
                        exp_date, mfg_date = exp_date, mfg_date
                    print("exp date set %s" % (exp_date))

                date_text = ''


    if not mfg_date:
        print ("Couldn't identify the manufacture date - Exiting..\n")
        return

    # default value : "Best before 12 months"
    if mfg_date and not exp_date:
        exp_date = mfg_date+relativedelta(months=+default_exp)
        print("Exp date set to defaults %s" % (exp_date))
    
    return (datetime.today() >= exp_date)


def is_harmful(content):

    global harm_elements
    content = content.split('\n')

    for line in content:
        line = line.lower()
        for key in harm_list:
            if line.find(key) != -1:
                harm_elements.append(key)

    if harm_elements:
        print("Harmful elements in this product are : ")
        print(harm_elements) 
        return True
    else:
        return False


def run_local(path):
    return detect_text(path)

def run_uri(uri):
    return detect_text_uri(uri)

def detect_text(path):
    """Detects text in the file."""
    from google.cloud import vision
    import io
    client = vision.ImageAnnotatorClient()

    # [START vision_python_migration_text_detection]
    with io.open(path, 'rb') as image_file:
        content = image_file.read()

    image = vision.Image(content=content)

    response = client.text_detection(image=image)
    texts = response.text_annotations

    content = texts[0].description
    return content

    #for text in texts:
    #    print('\n"{}"'.format(text.description))

    #   vertices = (['({},{})'.format(vertex.x, vertex.y)
    #               for vertex in text.bounding_poly.vertices])

    #   print('bounds: {}'.format(','.join(vertices)))

    if response.error.message:
        raise Exception(
            '{}\nFor more info on error messages, check: '
            'https://cloud.google.com/apis/design/errors'.format(
                response.error.message))

    # [END vision_python_migration_text_detection]

# [END vision_text_detection]


# [START vision_text_detection_gcs]
def detect_text_uri(uri):
    """Detects text in the file located in Google Cloud Storage or on the Web.
    """
    from google.cloud import vision
    client = vision.ImageAnnotatorClient()
    image = vision.Image()
    image.source.image_uri = uri

    response = client.text_detection(image=image)
    texts = response.text_annotations
    print('Texts:')

    for text in texts:
        print('\n"{}"'.format(text.description))

        vertices = (['({},{})'.format(vertex.x, vertex.y)
                    for vertex in text.bounding_poly.vertices])

        print('bounds: {}'.format(','.join(vertices)))

    if response.error.message:
        raise Exception(
            '{}\nFor more info on error messages, check: '
            'https://cloud.google.com/apis/design/errors'.format(
                response.error.message))
# [END vision_text_detection_gcs]



if __name__ == '__main__':
    parser = argparse.ArgumentParser(
        description=__doc__,
        formatter_class=argparse.RawDescriptionHelpFormatter)
    subparsers = parser.add_subparsers(dest='command')


    detect_text_parser = subparsers.add_parser(
        'exp', help="Identify is the product is expired - Local Image")
    detect_text_parser.add_argument('path')

    text_file_parser = subparsers.add_parser(
        'exp-uri', help="Identify if the product is expired - Image in Google Cloud Storage or  Web")
    text_file_parser.add_argument('uri')

    detect_text_parser = subparsers.add_parser(
        'harmful', help="Identify is the product has any harmful ingredients - Local Image")
    detect_text_parser.add_argument('path')

    text_file_parser = subparsers.add_parser(
        'harmful-uri', help="Identify if the product has any harmful ingredients - Image in Google Cloud Storage or  Web")
    text_file_parser.add_argument('uri')
    
    text_file_parser = subparsers.add_parser(
        'rest', help="run as a REST server")

    args = parser.parse_args()

    if 'rest' in args.command:
        app.run()

    if 'uri' in args:
        con = run_uri(args.uri)

    if 'path' in args:
        con = run_local(args.path)

    if 'exp' in args.command:
        result = is_expired(con)
        print("Expired = %s " % (result))

    if 'harmful' in args.command:
        result = is_harmful(con)
        print("\nHarmful = %s " % (result))

