<template>
  <div id="configurator">
    <div style="float:right;">
      <label for="snapshot">Snapshot</label>
      <br />
      <div style="height: 180px; border: 1px solid grey; display: block">
        <img id="snapshot" :src="snapshotUrl" style="height: 180px" v-if="snapshotUrl!==''" />
      </div>

      <p>
        <label for="cookie">Session cookie</label>
        <br />
        <input id="cookie" type="text" v-model="cookie" />
      </p>
      <p>
        <label for="responseText">Response</label>
        <br />

        <textarea name="responseText" id="responseText" cols="38" rows="10" v-model="responseText"></textarea>
        <br />
        <label for="settings">Settings</label>
        <br />
        <textarea name="settings" id="settings" cols="38" rows="10" v-model="settings"></textarea>
      </p>
      <div v-if="aps.length>0">
        <h4>Detected access points (APs)</h4>
        <span>Click to select</span>
        <ul>
          <li
            class="ssid"
            v-for="ap in aps"
            :key="ap.ssid"
            @click="selectSsid(ap)"
          >{{ap.ssid}} ({{ap.cipher}})</li>
        </ul>
      </div>
    </div>
    <h1>WLC-1000 Home Cam Mini configurator</h1>
    <h3>Two situations are possible:</h3>
    <ol>
      <li>Your camera has been factory reset to broadcast the 'Home Cam Mini' network, and you're connected to it</li>
      <li>Your camera has been preconfigured (when the Sitecom Mycam app still worked) to use your current network</li>
    </ol>

    <p>In case of (2), please input the IP-address of the camera here:</p>
    <input type="text" v-model="ip" />
    <button v-on:click="getSnapshot">Get snapshot</button>

    <h3>Steps to configure wifi:</h3>
    <ol>
      <li>Click 'Get wifi info'</li>
      <li>Change SSID and password</li>
      <li>Click 'Apply'</li>
      <li>Check the response at the right to see if the settings were changed</li>
      <li>Click 'Apply changes' (will reboot the camera)</li>
      <li>Access the streams with rtsp://{{ip}}/live1 (high quality) and rtsp://{{ip}}/live2 (low quality)</li>
    </ol>

    <h2>Actions</h2>

    <p>
      <button @click="getCloudModeInfo">Get wifi info</button>
      <button @click="applyChanges">
        <strong>Apply changes</strong>
      </button>
    </p>

    <p>
      <label for="ssid">SSID + cipher</label>
      <br />
      <input id="ssid" type="text" v-model="ssid" />
      <input id="cipher" type="text" v-model="cipher" placeholder="WPA, WPA2 or WEP" />
    </p>
    <p>
      <label for="password">Password</label>
      <br />
      <input id="password" type="text" v-model="password" />
      <button @click="setSsid">
        <strong>Apply</strong>
      </button>
    </p>
    <h2>Settings</h2>
    <p>
      <label for="motiondetection">Motion detection</label>
      <br />
      <input
        type="radio"
        name="motiondetection"
        id="motiondetection_on"
        value="ON"
        v-model="motiondetection"
      />
      <label for="motiondetection_on">On</label>
      <input
        type="radio"
        name="motiondetection"
        id="motiondetection_off"
        value="OFF"
        v-model="motiondetection"
      />
      <label for="motiondetection_off">Off</label>
      <button @click="setMotiondetection">
        <strong>Apply</strong>
      </button>
    </p>
    <p>
      <label for="audiodetection">Audio detection</label>
      <br />
      <input
        type="radio"
        name="audiodetection"
        id="audiodetection_on"
        value="ON"
        v-model="audiodetection"
      />
      <label for="audiodetection_on">On</label>
      <input
        type="radio"
        name="audiodetection"
        id="audiodetection_off"
        value="OFF"
        v-model="audiodetection"
      />
      <label for="audiodetection_off">Off</label>
      <button @click="setAudiodetection">
        <strong>Apply</strong>
      </button>
      <br />
      <button @click="setNtp">
        <strong>Set NTP</strong>
      </button>
      <br />
      <button @click="setOSD">
        <strong>Set OSD on</strong>
      </button>
    </p>
    <h2>Other tools</h2>
    <button @click="scanAP">Scan for access points</button>
    <button @click="cameraLogin">Get login cookie</button>
    <button @click="cameraLogout">Logout camera</button>
  </div>
</template>

<script>
const http = require("request").defaults({ jar: true, debug: true });

export default {
  name: "HelloWorld",
  data() {
    return {
      ip: "192.168.11.188",
      snapshotUrl: "",
      responseText: "",
      settings: "",
      cookie: "",
      ssid: "",
      cipher: "",
      password: "",
      aps: [],
      motiondetection: "ON",
      audiodetection: "ON"
    };
  },
  methods: {
    getSnapshot() {
      this.snapshotUrl = `http://${this.ip}/img/snapshot.jpg?${Date.now()}`;
    },
    cameraLogin() {
      http
        .post(`http://${this.ip}/CGI/CameraLogin`)
        .on("data", data => (this.responseText = data))
        .on(
          "response",
          response => (this.cookie = response.headers["set-cookie"])
        )
        // eslint-disable-next-line
        .on("error", error => console.log(error));
    },
    cameraLogout() {
      if (this.cookie !== "") {
        http
          .post(`http://${this.ip}/CGI/RemoteControl`, {
            headers: { Cookie: this.cookie, "Content-Type": "text/plain" },
            body: "<RemoteControl><Action>RESTART</Action></RemoteControl>"
          })
          // eslint-disable-next-line
          .on("response", response => {
            this.cookie="";
            this.getCloudModeInfo();
          });
      }
    },
    scanAP() {
      http
        .get(`http://${this.ip}/CGI/ScanAP`)
        .on("data", data => {
          this.aps = [];
          this.responseText = data;
          const parser = new DOMParser();
          let xmlDoc = parser.parseFromString(data, "text/xml");
          const elements = xmlDoc.getElementsByTagName("AP");
          for (let i = 0; i < elements.length; i++) {
            this.aps.push({
              ssid: elements[i].children[0].textContent,
              cipher: elements[i].children[2].textContent
            });
          }
          // this.aps = xmlDoc;
        })
        // eslint-disable-next-line
        .on("error", error => console.log(error));
    },
    selectSsid(ap) {
      this.ssid = ap.ssid;
      this.cipher = ap.cipher;
    },
    setSsid() {
      http
        .post(`http://${this.ip}/CGI/CameraLogin`)
        .on("data", data => (this.responseText = data))
        .on("response", response => {
          this.cookie = response.headers["set-cookie"];
          http.get(`http://${this.ip}/CGI/CloudmodeInfo`).on("data", data => {
            let cloudModeInfo = this.decode(data);
            cloudModeInfo = cloudModeInfo.replace(
              /<SSID>.*<\/SSID><PASSWORD>.*<\/PASSWORD><CIPHER>.*<\/CIPHER>/,
              `<SSID>${this.ssid}</SSID><PASSWORD>${this.password}</PASSWORD><CIPHER>${this.cipher}</CIPHER>`
            );
            const encoded = this.encode(cloudModeInfo);
            http
              .post(`http://${this.ip}/CGI/CloudmodeSetup`, {
                headers: { Cookie: this.cookie, "Content-Type": "text/plain" },
                body: encoded
              })
              // eslint-disable-next-line
              .on("response", response => {
                this.getCloudModeInfo();
              });
          });
        });
    },
    setMotiondetection() {
      const postString =
        this.motiondetection === "ON"
          ? "<Item><MOTION_DETECTION><ENABLE>ON</ENABLE><SENSITIVITY>3</SENSITIVITY></MOTION_DETECTION></Item>"
          : "<Item><MOTION_DETECTION><ENABLE>OFF</ENABLE><SENSITIVITY>0</SENSITIVITY></MOTION_DETECTION></Item>";
      http
        .post(`http://${this.ip}/CGI/CameraLogin`)
        .on("data", data => (this.responseText = data))
        .on("response", response => {
          this.cookie = response.headers["set-cookie"];
          http
            .post(`http://${this.ip}/CGI/CameraSetup`, {
              headers: { Cookie: this.cookie, "Content-Type": "text/plain" },
              body: postString
            })
            // eslint-disable-next-line
            .on("response", response => {
              this.getCameraSetting();
            });
        });
    },
    setNtp() {
      const postString =
        "<Item><TIME_ZONE><GMT>+1</GMT><NTP>router.lan</NTP></TIME_ZONE></Item>";
      http
        .post(`http://${this.ip}/CGI/CameraLogin`)
        .on("data", data => (this.responseText = data))
        .on("response", response => {
          this.cookie = response.headers["set-cookie"];
          http
            .post(`http://${this.ip}/CGI/CameraSetup`, {
              headers: { Cookie: this.cookie, "Content-Type": "text/plain" },
              body: postString
            })
            // eslint-disable-next-line
            .on("response", response => {
              this.getCameraSetting();
            });
        });
    },
    setOSD() {
      const postString =
        "<Item><LIGHT>OFF</LIGHT></Item><Item><OSD><Chan><Idx>0</Idx><Enable>ON</Enable><X>0.000000</X><Y>0.000000</Y><CNT>{Time}</CNT><Color>WHITE</Color><Size>1X</Size></Chan><Chan><Idx>1</Idx><Enable>ON</Enable><X>1.000000</X><Y>1.000000</Y><CNT>{Time}</CNT><Color>WHITE</Color><Size>1X</Size></Chan></OSD></Item>";
      http
        .post(`http://${this.ip}/CGI/CameraLogin`)
        .on("data", data => (this.responseText = data))
        .on("response", response => {
          this.cookie = response.headers["set-cookie"];
          http
            .post(`http://${this.ip}/CGI/CameraSetup`, {
              headers: { Cookie: this.cookie, "Content-Type": "text/plain" },
              body: postString
            })
            // eslint-disable-next-line
            .on("response", response => {
              this.getCameraSetting();
            });
        });
    },
    setAudiodetection() {
      const postString =
        this.motiondetection === "ON"
          ? "<Item><AUDIO_DETECTION><ENABLE>ON</ENABLE><SENSITIVITY>3</SENSITIVITY></AUDIO_DETECTION></Item>"
          : "<Item><AUDIO_DETECTION><ENABLE>OFF</ENABLE><SENSITIVITY>0</SENSITIVITY></AUDIO_DETECTION></Item>";
      http
        .post(`http://${this.ip}/CGI/CameraLogin`)
        .on("data", data => (this.responseText = data))
        .on("response", response => {
          this.cookie = response.headers["set-cookie"];
          http
            .post(`http://${this.ip}/CGI/CameraSetup`, {
              headers: { Cookie: this.cookie, "Content-Type": "text/plain" },
              body: postString
            })
            // eslint-disable-next-line
            .on("response", response => {
              this.getCameraSetting();
            });
        });
    },
    applyChanges() {
      //http://192.168.1.1/CGI/RemoteControl
      //<RemoteControl><Action>GOCLOUD</Action></RemoteControl>
      http
        .post(`http://${this.ip}/CGI/CameraLogin`)
        .on("data", data => (this.responseText = data))
        .on("response", response => {
          this.cookie = response.headers["set-cookie"];
          http
            .post(`http://${this.ip}/CGI/RemoteControl`, {
              headers: { Cookie: this.cookie, "Content-Type": "text/plain" },
              body: "<RemoteControl><Action>GOCLOUD</Action></RemoteControl>"
            })
            // eslint-disable-next-line
            .on("response", response => {
              this.getCloudModeInfo();
            });
        });
    },
    getCloudModeInfo() {
      http.get(`http://${this.ip}/CGI/CloudmodeInfo`).on("data", data => {
        const decoded = this.decode(data);
        this.responseText = decoded;
        const matches = decoded.match(
          /<SSID>(.*)<\/SSID><PASSWORD>(.*)<\/PASSWORD><CIPHER>(.*)<\/CIPHER>/
        );
        this.ssid = matches[1];
        this.password = matches[2];
        this.cipher = matches[3];
        //const encoded = this.encode(decoded)
        // eslint-disable-next-line
        //console.table(data, encoded);
        //this.cookie = data === encoded;
      });
    },
    getCameraSetting() {
      http
        .get(
          `http://${this.ip}/CGI/CameraSetting?MOTION_DETECTION&AUDIO_DETECTION&OSD&TIME_ZONE&TIME&LIGHT&NAME`
        )
        .on("data", data => {
          this.settings = data;
        });
    },
    decode(bytes) {
      let decoded = new Uint8Array(bytes.length - 2);
      for (let i = 0; i < decoded.length; i++) {
        decoded[i] = bytes[i] ^ 128;
      }
      return this.bin2String(decoded);
    },
    encode(input) {
      const bytes = this.string2Bin(input);
      let checksum = 0;
      let encodedBytes = new Uint8Array(bytes.length + 2);
      for (let i = 0; i < bytes.length; i++) {
        encodedBytes[i] = bytes[i] ^ 128;
        checksum = (encodedBytes[i] & 255) + checksum;
      }
      encodedBytes[encodedBytes.length - 1] = checksum & 255;
      encodedBytes[encodedBytes.length - 2] = (checksum >> 8) & 255;
      return Buffer.from(encodedBytes);
    },

    bin2String(array) {
      var result = "";
      for (let i = 0; i < array.length; i++) {
        result += String.fromCharCode(array[i]);
      }
      return result;
    },
    string2Bin(str) {
      var result = [];
      for (let i = 0; i < str.length; i++) {
        result.push(str.charCodeAt(i).toString());
      }
      return result;
    }
  }
};
</script>

<style scoped>
li.ssid:hover {
  cursor: pointer;
  text-decoration: underline;
}
</style>

