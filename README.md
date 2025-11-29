my-image-editor/
  ├── public/
  │     └── index.html
  ├── server.js
  └── package.json
  {
  "name": "my-image-editor",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.2",
    "multer": "^1.4.5",
    "axios": "^1.4.0",
    "form-data": "^4.0.0"
  }
}
const express = require('express');
const multer = require('multer');
const fs = require('fs');
const path = require('path');
const axios = require('axios');
const FormData = require('form-data');

const app = express();
const upload = multer({ dest: 'uploads/' });

const OPENAI_API_KEY = 'YOUR_OPENAI_API_KEY_HERE';

app.use(express.static('public'));

app.post('/edit-image', upload.single('image'), async (req, res) => {
  try {
    const imagePath = req.file.path;

    const form = new FormData();
    form.append('image', fs.createReadStream(imagePath));
    // If you want to use a mask, you would also append a 'mask' file here.
    form.append('n', 1);
    form.append('size', '1024x1024');
    form.append('model', 'dall-e-2');  // or whichever model
    form.append('response_format', 'url');
    form.append('prompt', req.body.prompt || '');  // you can allow user to send a prompt

    const response = await axios.post(
      'https://api.openai.com/v1/images/edits',
      form,
      {
        headers: {
          'Authorization': `Bearer ${OPENAI_API_KEY}`,
          ...form.getHeaders()
        }
      }
    );

    // Clean up uploaded file
    fs.unlinkSync(imagePath);

    const imageUrl = response.data.data[0].url;
    res.json({ imageUrl });
  } catch (error) {
    console.error(error.response?.data || error);
    res.status(500).json({ error: 'Image editing failed' });
  }
});

app.listen(3000, () => {
  console.log('Server listening on http://localhost:3000');
});
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>AI Image Editor</title>
  <style>
    body {
      margin: 0;
      font-family: Arial, sans-serif;
      background: linear-gradient(135deg, #89f7fe, #66a6ff);
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      flex-direction: column;
    }
    #container {
      background: rgba(255,255,255,0.2);
      padding: 30px;
      border-radius: 15px;
      text-align: center;
      backdrop-filter: blur(10px);
    }
    input[type="file"], input[type="text"] {
      padding: 10px;
      border-radius: 8px;
      border: none;
      cursor: pointer;
      margin-bottom: 15px;
      width: 200px;
    }
    button {
      padding: 10px 20px;
      border-radius: 8px;
      border: none;
      cursor: pointer;
      background-color: #fff;
      font-weight: bold;
    }
    img {
      margin-top: 20px;
      max-width: 300px;
      border-radius: 10px;
    }
  </style>
</head>
<body>
  <div id="container">
    <h1>Upload Photo</h1>
    <input type="file" id="imageInput" accept="image/*"><br>
    <input type="text" id="promptInput" placeholder="Edit prompt (optional)"><br>
    <button onclick="uploadAndEdit()">Edit Image</button>
    <div id="result"></div>
  </div>

  <script>
    async function uploadAndEdit() {
      const input = document.getElementById('imageInput');
      const prompt = document.getElementById('promptInput').value;

      if(!input.files || !input.files[0]) {
        alert('Please upload an image first!');
        return;
      }

      const formData = new FormData();
      formData.append('image', input.files[0]);
      formData.append('prompt', prompt);

      const resp = await fetch('/edit-image', {
        method: 'POST',
        body: formData
      });

      const data = await resp.json();
      if(data.error) {
        alert('Editing failed: ' + data.error);
      } else {
        document.getElementById('result').innerHTML = `<img src="${data.imageUrl}" alt="edited image">`;
      }
    }
  </script>
</body>
</html>
