<template>
  <el-container class="content-container">
    <el-header class="header">


      File Encryption/Decryption Program

      <el-popover
        popper-class="popover-text"
        placement="top-start"
        title="File Encrypter/Decrypter"
        :width="200"
        trigger="hover"
      >
        <template #reference>
          <i class="el-icon-info"></i>
        </template>
          <span>By dropping a file in the leftmost box and clicking "Encrypt & Download", you may encrypt and download a file. In the same manner, by dropping a file into the rightmost box and clicking "Decrypt & Download" an encrypted file may be decrypted and downloaded</span>
      </el-popover>

    </el-header>

    <el-main class="file-space-container">
      <div class="encryption-key-area"></div>


        <div class="file-drop-container">
          <div class="file-drop-location" @drop.prevent="dropFile_encrypt" @dragover.prevent>
            <span v-if="!encryptionFile">Drop a file to be encrypted here</span>
            <div v-else class="file-description-box">
              <div>File Name: {{ fileInfo.fileName }}</div>
              <div>File Type: {{ fileInfo.fileType }}</div>
              <div>File Size: {{ calcFileSize_encrypt }}</div>
            </div>
          </div>


          <div class="encryption-key-container">
            Encryption Key: {{ key }}
            <el-popover
            popper-class="popover-text"
            placement="top-start"
            title="File Encrypter/Decrypter"
            :width="200"
            trigger="hover"
            >
            <template #reference>
              <i class="el-icon-info"></i>
            </template>
              <span>This is your encryption key - save it and use it to decrypt your encrypted file</span>
          </el-popover>
          </div>
        </div>

        <div class="file-drop-container">

          <div class="file-drop-location" @drop.prevent="dropFile_decrypt" @dragover.prevent>
            <span v-if="!decryptionFile">Drop file to be decrypted here</span>
            <div v-else class="file-description-box">
              <div>File Name: {{ fileContents.fileName }}</div>
              <div>File Type: {{ fileContents.fileType }}</div>
              <div>File Size: {{ calcFileSize_decrypt }}</div>
            </div>
          </div>

          <el-input class="decryption-key-input" size="mini" placeholder="Please put encryption key here..." v-model="decryptionKey"></el-input>

        </div>


    </el-main>
    <el-footer class="button-container">
      <el-button type="primary" @click="GCM_encrypt">Encrypt & Download</el-button>
      <el-button type="primary" @click="GCM_decrypt">Decrypt & Download</el-button>
    </el-footer>
  </el-container>
</template>


<script>

export default {
 data() {
   return {
     //Booleans to track if a file has been added
     encryptionFile: false,
     decryptionFile: false,

     plainFile: null,
     decodedFile: null,
     encodedFile: null,
     file_to_decrypt: null,
     fileInfo: {
       data: null,
       fileName: null,
       fileType: null,
       fileSize: null,
       iv: null
     },
     fileContents: {
       data: null,
       fileName: null,
       fileType: null,
       fileSize: null,
       iv: null
     },

     decryptionKey: '',
     key: '',
     keyData: null
   }
 },
 computed: {
   calcFileSize_encrypt() {
     let size = this.fileInfo.fileSize;
     if(size > 1000000)
     {
       return (size / 1000000) + ' MB';
     }
     else
     {
      return (size / 1000) + ' kB';
     }
   },
   calcFileSize_decrypt() {
     let size = this.fileContents.fileSize;
     if(size > 1000000)
     {
       return (size / 1000000) + ' MB';
     }
     else
     {
      return (size / 1000) + ' kB';
     }
   }
 },
 methods: {
   // ===============================================================================================================================================
   // =========================================================== Encryption Functions ==============================================================
   // ===============================================================================================================================================
   dropFile_encrypt(e) { // called on drop event in encryption div
     this.key = this.generateHexString(64);
     this.keyData = Uint8Array.from(Buffer.from(this.key, 'hex'));
     this.encryptionFile = true;
     let file = e.dataTransfer.files[0]; // file starts as a file
     let reader = new FileReader();

     this.fileInfo.fileType = file.type; // storing file type to be packaged with the file for decryption use
     this.fileInfo.fileName = file.name;
     this.fileInfo.fileSize = file.size;

     reader.readAsDataURL(file); // read in the file as encoded base64
     reader.onload = (e) => {
       if(e.target.readyState == FileReader.DONE)
       {
         let arrayBuffer = e.target.result;
         arrayBuffer = arrayBuffer.substr(arrayBuffer.indexOf('base64,') + 7);
         this.plainFile = new Uint8Array(atob(arrayBuffer).split("").map((char)=>char.charCodeAt(0)));
       }
     }

    },
   GCM_encrypt() {
     this.fileInfo.iv = crypto.getRandomValues(new Uint8Array(16)); //setting the nonce (will be packaged with the file later)
     
     crypto.subtle.importKey("raw", this.keyData, "aes-gcm", false, ['encrypt'])
     .then((key) => {
       return crypto.subtle.encrypt({name: "aes-gcm", iv: this.fileInfo.iv}, key, this.plainFile); // at this point, this.plainFile is a uint8array, so it should be fine to pass as-is into the encrypt() function
     })
     .then((cipherText) => {
       this.encodedFile = Buffer.from(cipherText).toString('hex'); //convert cipherText of ArrayBuffer to hex string for JSON compatibility
       this.downloadFile_Encryption();
     });
    },
   downloadFile_Encryption() {
     this.fileInfo.data = this.encodedFile;
     this.fileInfo.iv = btoa(String.fromCharCode.apply(null, this.fileInfo.iv)); // before packaging all the information together, since Uint8Array is not supported by JSON, we encode the IV to base64
     
     
     let realFile = JSON.stringify(this.fileInfo); // Stringify me up, Scotty!

     const file = new Blob([realFile], {type: 'text/plain'});

     // ==== create invisible anchor tag and programatically 'click' it for download: =====
     let downloadLink = document.createElement("a");
     downloadLink.href = window.URL.createObjectURL(file);
     downloadLink.download = "myencrypted.file";
     downloadLink.click();
     // ===================================================================================
    },
   // ===============================================================================================================================================




   // ===============================================================================================================================================
   // ========================================================= Decryption Functions ================================================================
   // ===============================================================================================================================================
   dropFile_decrypt(e) { // called on drop event in decryption div
     this.decryptionFile = true;
     let file = e.dataTransfer.files[0];
     let reader = new FileReader();

     reader.readAsText(file); // file received is read as text for the JSON.parse later
     
     reader.onload = (e) => {
       if(e.target.readyState == FileReader.DONE)
       {         
         // ===== unpacking the IV seperately here, because it needs to be re-processed into a readable BufferSource format ======
         let iv = JSON.parse(e.target.result).iv;
         let uint8iv = new Uint8Array(atob(iv).split("").map((char)=>char.charCodeAt(0))); // decode iv from base64 back to Uint8Array, to be read by the subtlecrypto API
         // ======================================================================================================================

         // ==== supplying to the internal decryption data variable, the values recieved from the incoming file to decrypt ====
         this.fileContents.data = JSON.parse(e.target.result).data; // data is recieved in hex form
         this.fileContents.fileType = JSON.parse(e.target.result).fileType;
         this.fileContents.fileName = JSON.parse(e.target.result).fileName;
         this.fileContents.fileSize = JSON.parse(e.target.result).fileSize;
         this.fileContents.iv = uint8iv;
         // ===================================================================================================================
         
         // ===== moving the .data attribute to a different variable for the decryption process =====
         let hexFile = this.fileContents.data;
         this.file_to_decrypt = Uint8Array.from(Buffer.from(hexFile, 'hex'));
         // =========================================================================================
       }
     }
    },
   GCM_decrypt() {
     if(this.decryptionKey.length < 64)
     {
       this.openErrorBox('Please check your encryption key;', 'Encryption keys should be 64 characters in length.')
     }
     let KeyData = Uint8Array.from(Buffer.from(this.decryptionKey, 'hex'))
     crypto.subtle.importKey("raw", KeyData, "aes-gcm", false, ['decrypt']) //replace KeyData with this.keyData if it doesn't work
     .then((key) => {
       return crypto.subtle.decrypt({name: "aes-gcm", iv: this.fileContents.iv}, key, this.file_to_decrypt); // this.file_to_decrypt should already be of type ArrayBuffer, so it should be good to send in as-is
     })
     .then((plainFile) => {
       let stringifiedData = new Uint8Array(plainFile).reduce((data, byte) => { // decoding the ArrayBuffer to a string
         return data + String.fromCharCode(byte);
        }, '');
       this.decodedFile = btoa(stringifiedData); // re-encoding the string into base64

      //  this.decodedFile = btoa(String.fromCharCode.apply(null, new Uint8Array(plainFile))); // encoding into base64 from BufferSource

       this.downloadFile_Decryption();
     })
     .catch((e) => {
       console.log('Caught Error: ' + e); // here because no-unused-variables also logging errors to console
       if(!this.fileContents.data)
       {
         this.openErrorBox('Decryption File Missing', 'Please input a file to decrypt.')
       }
       else if(this.decryptionKey.length == 64)
       {
        this.openErrorBox('Decryption Key is Incorrect', 'Please check your decryption key to make sure that it is correct.');
       }
     });
    },
   downloadFile_Decryption() {
     let fileType = this.fileContents.fileType;
     let fileName = this.fileContents.fileName;
     const decodedFile = this.dataURLtoBlob(this.decodedFile);
     const file = new Blob([decodedFile], {type: fileType})

     // ==== create invisible anchor tag and programatically 'click' it for download: =====
     let downloadLink = document.createElement("a");
     downloadLink.href = window.URL.createObjectURL(file);
     downloadLink.download = fileName;
     downloadLink.click();
     // ===================================================================================
    },
    // ===============================================================================================================================================


    // ==================================================================================================================================
    // =================================================== Helper Functions =============================================================
    // ==================================================================================================================================
    dataURLtoBlob(dataurl) {
        let bstr = atob(dataurl);
        let n = bstr.length, u8arr = new Uint8Array(n);
    while(n--){
        u8arr[n] = bstr.charCodeAt(n);
    }
    return u8arr;
    },
    generateHexString(length) {
      var ret = "";
      while (ret.length < length) {
        ret += Math.random().toString(16).substring(2);
      }
      return ret.substring(0,length);
    },
    openErrorBox(title, msg) {
      this.$alert(msg, title, {
        confirmButtonText: 'OK',
      });
    }
   // ==================================================================================================================================
  }
}
</script>


<style lang="less">
* { // CSS reset
  position: block;
  margin: 0;
  padding: 0;
  border: 0;
  outline: 0;
  font-size: 100%;
  vertical-align: baseline;
  background: transparent;
}

.el-icon-info {
  margin-left: 5px;
}


.el-popover {
  word-break: normal !important;
  width: 350px !important;
}


.content-container {
  display: flex;
  flex-direction: column;

  height: 90vh;


  .header {
    display: flex;
    justify-content: center;
    align-items: center;
    
    background-color: #fafafa;

    border-bottom: 1px solid #ebebed
  }


  .file-space-container {
    display: flex;
    flex-direction: row;

    background-color: #ffff;


    .file-drop-container {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;

      width: 50%;
      height: 100%;


      .encryption-key-container {
        display: flex;
        align-items: center;
        justify-content: space-between;

        margin-top: 15px;
        padding: 5px;

        height: 25px;

        background-color: #fafafa;

        border: solid 1px #ebebed;
        border-radius: 4px;

        font-size: 0.8em;
      }


      .decryption-key-input {
        
        height: 20px;
        width: 450px;

        margin-top: 15px;


        .el-input__inner {
          height: 25px;
          padding-top: 5px;
          padding-bottom: 5px;
        }
      }

    }


    .file-drop-location {
      display: flex;
      align-items: center;
      justify-content: center;

      width: 30vw;
      height: 65vh;

      background-color: #fafafa;

      border: dashed 1px #ebebed;
      border-radius: 4px;
      box-shadow: 0 2px 4px rgba(0, 0, 0, .12), 0 0 6px rgba(0, 0, 0, .04);
    }


    .dropLocation {
      width: 500px;
      height: 500px;
    }
  }


  .button-container {
    display: flex;
    flex-direction: row;
    justify-content: space-around;
  }
}
</style>
