<template>
 <div>
   
  <div  >
    <h6> Control of Power Supply with SCPI over WebUSBTMC</h6>
    <div class="q-gutter-md">
      <div class="row q-gutter-md ">
        <div class="col-xs-12 col-sm-6 col-md-4">
          <q-card >
            <q-card-section class="q-gutter-md">
              <div class="text-h6">Options</div>
              <div class="text-subtitle2">Choose units to accumulate</div>
              <q-option-group
                v-model="group"
                :options="accumUnits"
                color="green"
                left-label
                inline
              />
             
              <input type="text" ref="accumQuant" class="form-control" placeholder="Enter stop point" /> {{group}}
              <br> Read Interval <input type="number" v-model="readInterval" /> ms 
            <q-btn ref="startBtn"  @click="startBtnClick()"> Start</q-btn>
            </q-card-section>
          </q-card>
        </div>
        
        <div class="col-xs-12 col-sm-6 col-md-4 ">
          <q-card >
            <q-card-section class="q-gutter-md" >
              <div class="text-h6">Connect Device</div>
              <q-btn ref="connectButton"  @click="connectButtonClick()"> {{connectionState}} </q-btn> <output ref="devName" ></output>
              <br>
              <output id="notification-message">Notifications</output>
            </q-card-section>
          </q-card>
        </div>
        
      </div>
      <div class="row">
        <div class="col-sm-12 col-md-8">
          <q-card class="q-gutter-md">
            <div class="text-h6">Manual Commands</div>
            <q-card-section>
              <div class="row">
                <input type="text" ref="inputText" class="form-control" placeholder="Select or Enter Command" >  
                <q-btn id="wr-button" @click="writeBtnClick()">Write</q-btn> 
              </div>
            
            </q-card-section>
            
            <q-card-section>
              <div class="row">
                <div class="col">
                    <div class="row">
                       <q-btn id="rd-button" @click="read()">Read</q-btn>
                    </div>
                      <div class="row">
                        <textarea class="form-control" v-model="outputTextArea" rows="5" ></textarea>
                      </div>
                </div>
              </div>
            </q-card-section>
          </q-card>
        </div>
      </div>
    </div>
  </div>
 
 </div>
</template>

<script>
// NOTE: use vanilla js version from webusbtmc folder as the source of code for this app
import Webusbtmc from '../assets/webusbtmc.js';


let tmc = new Webusbtmc();


//console.log("tmc is " , tmc);

const OutputTo = {  // in case biary or image data are needed (not implemented here)
  Text: 1,
  Image: 2,
  Download: 3,
};
let myOutputTo;

myOutputTo = OutputTo.Text;  // sets myOutputTo to 1 for display 



export default {
  name: 'PageIndex',
  data(){
    return {
     //tmc: {},
      group: '  -',
      accumUnits: [
        {
          label: 'Seconds',
          value: 'S'
        },
        {
          label: 'Coulombs',
          value: 'C'
        }
      ],
      isConnected: false,
      outputTextArea : "",
      accumCoulombs : true,
      accumulatedUnits : 0,
      stopPoint : 10,  //default is seconds
      startTime : 0,
      readInterval : 3000,
      cmdDelay : 200,  // milliseconds between commands
      elapsedTime : 0,
      resultArray: [],
      csvData : "",
      lastRead: "",
      readTimer: null
     // connectionState: "connected"
    }
  },
  computed: {
      connectionState() { 
        return this.isConnected ? "Disconnect" : "Connect"
        }
  },
  methods: {
 /*    setup() {
      tmc = new Webusbtmc();
      this.$refs.devName.innerHTML = "Connecting to " + JSON.stringify(tmc);
    }, */
    connectButtonClick() {
        console.log("in connectButtonClick() function");
        if (this.isConnected) {
            this.notify("");
            tmc.close();
            this.isConnected = false;
            this.$refs.devName.innerHTML = "    not connected";
            //this.$refs.connectButton.label = "Connect";
        } else {  // isConnected is true
            const filters = [
                { classCode: 0xfe, subclassCode: 0x03, protocolCode: 0x01 },
            ];
            navigator.usb
                .requestDevice({ filters: filters })
                .then((result) => {
                this.notify("");
                this.connect(result);
                })
                .catch((error) => {
                this.notify(error);
            });
        }
    },
    startBtnClick(){
      this.readTimer = setInterval(this.readAll, this.readInterval);
      
      let d = new Date()
      this.startTime = d.getTime()
        
    },
    async readAll( ){
      let text = "*READALL?";
      console.log("Text is ", text, " read at  ", this.elapsedTime, "ms after a delay of ", this.cmdDelay)
      tmc.write(text).then(() => {
                this.addOutput(text);
              }).catch((error) => {
                this.notify(error);
              });  
      
      setTimeout(this.readAllResponse, this.cmdDelay)
     
    },
    async readAllResponse(){
      let resultText = await this.read();
      this.resultArray.push(this.readAllParse(resultText));
              
      let d = new Date()
      let now = d.getTime()
      this.elapsedTime = now - this.startTime;
      this.accumulatedUnits =  this.elapsedTime / 1000;  //seconds
      
      console.log("l161 accum is ", this.accumulatedUnits, "  resultArray is ", this.resultArray);
      // this.accumulatedUnits = this.accumulatedUnits ;
      if (this.accumulatedUnits > this.stopPoint) {
        clearInterval(this.readTimer)
      };
      this.notify("all done!");
      this.compileData()
    },
    compileData(){
      console.log(this.resultArray);
    },
    readAllParse(resultStr = "1,2") {  // expects a string of numbers
      
      console.log("result String is ", resultStr)  
      let resultArray = resultStr.split(',');
      console.log("result array is ", resultArray);
      let d = new Date()
      let now = d.getTime()
      resultArray.unshift(now);
     return  resultArray; 
    },
    writeBtnClick() {
        console.log("in writeBtnClick() function ", this.isConnected);
        if (this.isConnected) {
            const text = this.$refs.inputText.value;
            /* if data is binary
                
            if ($writeWhich.html() == 'Binary Data' && binaryFile !== undefined && binaryFile !== null) {
                const buffer = new Uint8Array(binaryFile);

                if (text === undefined || text === '') {
                tmc.writeBytes(buffer).then(() => {
                    print('Data chunk was sent successfully');
                    $writeButton.html('Write');
                }).catch((error) => {
                    notify(error);
                    $writeButton.html('Write');
                });  
                }
                else {
                tmc.writeBlockData(text, buffer).then(() => {
                    print('Binary data was sent successfully');
                    $writeButton.html('Write');
                }).catch((error) => {
                    notify(error);
                    $writeButton.html('Write');
                });  
                }
            } */

            if (text === undefined || text === '') {
              this.notify('Text is empty');
            }
            else {
              tmc.write(text).then(() => {
                this.addOutput(text);
              }).catch((error) => {
                this.notify(error);
              });  
            }
        } else {
            this.notify('');
        }
      
    },

    read() {
        let resultText = "";
        console.log("in read() method", this.isConnected);
        if (this.isConnected) {
            this.notify("");
            let length = 1024;

            tmc
                .readBytes(length)
                .then((result) => {
                    console.log(" after tmc read, result is ", result);
                    if (!result || result.length == 0) {
                      this.addOutput("Failed to receive the response");
                    } else {
                        
                      if (typeof result === 'string' || result instanceof String) {
                        resultText = result;
                        console.log("result was string ", result);
                      }
                      else {
                        //console.log("result wasn't string ", result);
                        const textDecoder = new TextDecoder();
                        resultText = textDecoder.decode(result);
                        //console.log("resultText before replacement >", resultText, "<");
                        resultText = resultText.replace(tmc.terminatorRead,'');
                        //console.log("resultText after replacement >", resultText, "<");
                      } 
                      console.log("resultText is ", resultText);
                      this.addOutput(resultText);
                      this.lastRead = resultText;
                    }
                })
                .catch((error) => {
                this.notify(error);
                });
        }
        return resultText;
    },
    addOutput(text) {   // input must be a string
      if (!text) {
        return;
      }
      console.log("in addOutput, text is ", text)
      this.outputTextArea = this.outputTextArea + '\n' + text;
    
    },
    
    objectToCsv(data){
      // get the headers from the first spot
      //const headers = Object.keys(data[0]);
      console.log("objtocsv data ", data);
      const headers = Object.keys(data[0]);  // an array
      const firstRow = 'Row,' + headers.join(',');
      const csvRows = [];
      csvRows.push(firstRow);

      // loop over rows
      for (const [i, row] of data.entries() ){
        const values = headers.map(header => {
          // escape any internal commas on fields converted to string (by ''+)
        const escaped =(''+ row[header]).replace(/"/g, '\\"');
        return  escaped;   //'"${escaped}"';
        });
        let rowdata = i + ',' + values.join(',');
        csvRows.push(rowdata);
      }
      return csvRows.join('\n');
    },
    
    downloadData(){
      
      const timestamp = Date.now();
      const fileName = timestamp + '.csv';       
      const downloadCsv = function(data, file) {
        const blob = new Blob([data], { type: 'text/csv' });
        const url = window.URL.createObjectURL(blob);
        const a = document.createElement('a');  // a blank element to generate the download url
        a.setAttribute('hidden', '');
        a.setAttribute('href', url);
        
        a.setAttribute('download', file);
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
      }; 
      downloadCsv(this.csvData, fileName);
      //this.downloadImage();
    },

    connect(device) {
      this.$refs.devName.innerHTML = "    Connecting to " + device.productName + "...";
      tmc
        .open(device)
        .then(() => {
          this.deviceName = device.productName.replace(/\s/g, "");
          this.$refs.devName.innerHTML = "    " + device.productName + " connected.";
          this.isConnected = true;
          //this.$refs.connectButton.label = "Disconnect";
        })
        .catch((error) => {
          this.$refs.devName.innerHTML = "-";
          this.notify(error);
        });
    },

    notify(message) {
        document.getElementById("notification-message").innerHTML = "Notifications: " + message;
    }

  }
}
</script>
