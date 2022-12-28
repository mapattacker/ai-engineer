# Demo Site

For every model, a demo site should be created to demonstrate the reliability and use of the service, especially to product owners & clients. [Streamlit](https://www.streamlit.io/) is an amazing python library used to create ML demo sites fast, while providing a beautiful & consistent template.

To facilitate deployment, we should always ensure both `requirements.txt` & `Dockerfile` are created & tested. 

Below is an example how we can develop a simple object detection demo site.


```python
"""streamlit server for demo site"""

import json
import time

import requests
import streamlit as st
from PIL import Image

from utils_image import encode_image, draw_on_image, json2array_yolo


# streamlit settings
st.set_page_config(page_title='Demo Site')


json_data = \
{
  "requests": [
    {
      "features": [
        {
          "maxResults": 20,
          "min_height": 0.03,
          "min_width": 0.03,
          "score_th": 0.3,
          "nms_iou": 0.4,
        }
      ],
      "image": {
        "content": None
      }
    }
  ]
}


def hide_navbar():
    """hide navbar so its not apparent this is from streamlit"""
    hide_streamlit_style = """
    <style>
    #MainMenu {visibility: hidden;}
    footer {visibility: hidden;}
    </style>
    """
    st.markdown(hide_streamlit_style, unsafe_allow_html=True) 



def send2api(api, image, json_data, token, \
             maxfeatures, min_height, min_width, \
             score_th, nms_iou):
    """Sends JSON request & recieve a JSON response
    
    Args
    ----
    api (str): API endpoint
    image (image file): opened image file
    json_data (dict): json request template
    token (str): API token
    maxfeatures (int): max no. of objects to detect in image
    min_height (float): min height of bounding box (relative to H) to be included
    min_width (float): min width of bounding box (relative to W) to be included
    score_th (float): min prediciton score for bounding box to be included
    nms_iou (float): intersection over union, for non-max suppression
    
    Returns
    -------
    json_response (dict): API response
    """
    base64_bytes = encode_image(image)

    token = {"X-Bedrock-Api-Token": token}
    
    json_data["requests"][0]["image"]["content"] = base64_bytes
    json_data["requests"][0]["features"][0]["maxResults"] = maxfeatures
    json_data["requests"][0]["features"][0]["min_height"] = min_height
    json_data["requests"][0]["features"][0]["min_width"] = min_width
    json_data["requests"][0]["features"][0]["score_th"] = score_th
    json_data["requests"][0]["features"][0]["nms_iou"] = nms_iou

    response = requests.post(api, headers=token, json=json_data)
    # response = requests.post(api, json=json_data)
    json_response = response.content.decode('utf-8')
    json_response = json.loads(json_response)
    return json_response


def main(api):
    """design streamlit fronend"""
    st.title("SafetyCone Detection")

    token = st.text_input("API Token", type="password")
    uploaded_file = st.file_uploader("Upload an image.")

    if uploaded_file is not None and api != "":
        image = Image.open(uploaded_file)

        # header
        st.subheader("Uploaded Image")
        st.image(image, width=400)

        # sidebar
        st.sidebar.title("Change Parameters")
        maxfeatures = st.sidebar.slider("Max Features", min_value=1, max_value=50, value=20, step=1)
        min_height = st.sidebar.slider("Min Height", min_value=0.01, max_value=0.05, value=0.03, step=0.01)
        min_width = st.sidebar.slider("Min Width", min_value=0.01, max_value=0.05, value=0.03, step=0.01)
        score_th = st.sidebar.slider("Score Th", min_value=0.1, max_value=0.5, value=0.3, step=0.1)
        nms_iou = st.sidebar.slider("NMS IOU", min_value=0.1, max_value=0.5, value=0.4, step=0.1)

        if st.button("Send API Request"):
          st.title("Results")

          # send request
          start_time = time.time()
          json_response = send2api(api, image, json_data, token, \
                                   maxfeatures, min_height, min_width, \
                                   score_th, nms_iou)
          latency = time.time() - start_time
          st.write("**Est. latency = `{:.3f} s`**".format(latency))

          # result image
          st.subheader("Visualize Output")
          bboxes_json = json_response["safetycone"]["boundingPoly"]["normalizedVertices"]
          bboxes_array = json2array_yolo(bboxes_json)
          class_mapper = {0: "safetycone"}
          image_res = draw_on_image(image, bboxes_array, class_mapper)
          st.image(image_res, width=400)

          # api response
          st.subheader("API Response")
          st.json(json.dumps(json_response, indent=2))


if __name__ == "__main__":
    api = "http://localhost:5000"
    hide_navbar()
    main(api)
```

To launch the app, use `streamlit run app.py`.

![](https://github.com/mapattacker/ai-engineer/blob/master/images/streamlit1.png?raw=true)

After uploading a picture, the results are shown as such.

![](https://github.com/mapattacker/ai-engineer/blob/master/images/streamlit2.png?raw=true)