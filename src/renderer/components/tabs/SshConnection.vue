<template>
  <div>
    <div id="serverInfo">
      {{instance.name}}
      <span title="instance id">: {{instance.instanceId}}</span>
      <span v-if="instance.publicIp" title="public ip">: {{instance.publicIp}}</span>
      <span v-if="instance.privateIp" title="private ip">: {{instance.privateIp}}</span>
      <span title="instance type">: {{instance.instanceType}}</span>
      <span>: Font Size:
        <i class="fa fa-plus" title="Increase font size" v-on:click="fontSize('+')"></i>
        <i class="fa fa-minus" title="Decreate font size" v-on:click="fontSize('-')"></i>
      </span>
    </div>
    <div :id="'terminal' + tab" class="terminal"></div>
  </div>
</template>
<<script>
import * as Terminal from 'xterm';
import { ipcRenderer } from 'electron';
import os from 'os';
import * as pty from 'node-pty';
import path from 'path';
import fs from 'fs-extra';

const {app} = require('electron').remote;
// const Client = require('ssh2').Client

export default {
  name: 'sshConnection',
  data: function() {
    return {
      rendered: false,
      term: null,
      tab: null,
      connected: false,
      stream: null,
      org: null,
      cloudService: null,
      ptyProcess: null,
      instance: {}
    };
  },
  methods: {
    fontSize: function(dir) {
      let fontSize = parseInt($('#terminal' + this.tab + ' .terminal').css('font-size').replace(/px/, ''));
      if (dir === '+') {
        fontSize += 1;
      } else {
        fontSize -= 1;
      }
      $('#terminal' + this.tab + ' .terminal').css('font-size', fontSize + 'px');
      this.resizeIt();
    },
    resizeIt: function() {
      try {
        var term = this.term;
        var parentElementStyle = window.getComputedStyle(term.element.parentElement);
        var parentElementHeight = parseInt(parentElementStyle.getPropertyValue('height'));
        var parentElementWidth = Math.max(0, parseInt(parentElementStyle.getPropertyValue('width')));
        if (!$('#leftMenuExpanded').is(':visible')) {
          $('#terminal' + this.tab + ' .terminal').css('width', 'calc(100vw - 50px)');
          parentElementWidth += 200;
        } else {
          $('#terminal' + this.tab + ' .terminal').css('width', 'calc(100vw - 250px)');
        }
        var elementStyle = window.getComputedStyle(term.element);
        var elementPaddingVer = parseInt(elementStyle.getPropertyValue('padding-top')) + parseInt(elementStyle.getPropertyValue('padding-bottom'));
        var elementPaddingHor = parseInt(elementStyle.getPropertyValue('padding-right')) + parseInt(elementStyle.getPropertyValue('padding-left'));
        var availableHeight = parentElementHeight - elementPaddingVer;
        var availableWidth = parentElementWidth - elementPaddingHor;
        var subjectRow = term.rowContainer.firstElementChild;
        var contentBuffer = subjectRow.innerHTML;
        var characterHeight;
        var rows;
        var characterWidth;
        var cols;

        subjectRow.style.display = 'inline';
        subjectRow.innerHTML = 'W'; // Common character for measuring width, although on monospace
        characterWidth = subjectRow.getBoundingClientRect().width;
        subjectRow.style.display = ''; // Revert style before calculating height, since they differ.
        characterHeight = subjectRow.getBoundingClientRect().height;
        subjectRow.innerHTML = contentBuffer;

        rows = parseInt(availableHeight / characterHeight) - 3;
        cols = parseInt(availableWidth / characterWidth);
        term.resize(cols, rows);
        this.ptyProcess.resize(cols, rows);
        term.refresh(0, term.rows - 1);
      } catch (err) {
      }
    },
    ssh: async function (server) {
      const vue = this;
      /**
       *
        theme: {
        foreground: '#ffffff',
        background: '#000',
        cursor: '#ffffff',
        selection: 'rgba(255, 255, 255, 0.3)',
        black: '#000000',
        red: '#e06c75',
        brightRed: '#e06c75',
        green: '#A4EFA1',
        brightGreen: '#A4EFA1',
        brightYellow: '#EDDC96',
        yellow: '#EDDC96',
        magenta: '#e39ef7',
        brightMagenta: '#e39ef7',
        cyan: '#5fcbd8',
        brightBlue: '#5fcbd8',
        brightCyan: '#5fcbd8',
        blue: '#5fcbd8',
        white: '#d0d0d0',
        brightBlack: '#808080',
        brightWhite: '#ffffff'
      }
       */
      this.term = new Terminal({
      });
      this.term.open(document.getElementById('terminal' + this.tab), true);
      this.term._initialized = true;

      const appData = app.getPath('home');
      let keyPath = '';
      let keyCheck = path.join(appData, '.opshell', this.org.name, this.cloudService.identifier, server.user).replace(/\s+/g, '-');
      if (!fs.existsSync(keyCheck)) {
        keyCheck = path.join(appData, '.opshell', this.org.name, this.cloudService.identifier, server.instance.keyFile).replace(/\s+/g, '-');
      }
      if (fs.existsSync(keyCheck)) {
        keyPath = ' -i ' + keyCheck;
      }
      var shell = os.platform() === 'win32' ? 'powershell.exe' : 'bash';

      this.ptyProcess = pty.fork(shell, [], {
        name: 'xterm-color',
        cols: 80,
        rows: 10,
        cwd: process.env.HOME,
        env: process.env
      });
      if (!server.bastionHost) {
        const serverCommand = 'ssh -o "UserKnownHostsFile /dev/null" -o StrictHostKeyChecking=no ' + keyPath + ' -l ' + server.user + ' ' + server.ip;
        this.ptyProcess.write(serverCommand + '\r\n');
      } else {
        const bastionKeyPath = ' -i ' + path.join(appData, '.opshell', this.org.name, this.cloudService.identifier, server.bastionHost.keyFile).replace(/\s+/g, '-');
        const bastionCommand = 'ssh -t -o "UserKnownHostsFile /dev/null" -o StrictHostKeyChecking=no -o ProxyCommand=\'ssh ' + bastionKeyPath + ' -l ' + this.cloudService.bastionUser + ' ' + server.bastionHost.publicIp + ' nc %h %p\' ' + keyPath + ' -l ' + server.user + ' ' + server.ip;
        this.ptyProcess.write(bastionCommand + '\r\n');
      }
      this.ptyProcess.on('data', function(data) {
        vue.term.write(data);
      });

      vue.term.on('data', function(data) {
        vue.ptyProcess.write(data);
      });

      // This is hacky.   I can't figure out an event to know when to do this resize
      // so just going to wait a couple of seconds for now
      setTimeout(() => {
        this.resizeIt();
      }, 2000);

      /*
      var conn = new Client()
      conn.on('ready', function() {
        conn.shell(function(err, stream) {
          vue.stream = stream
          if (err) {
            console.log(err)
          }
          stream.on('close', function() {
            // console.log('Stream :: close')
            conn.end()
          }).on('data', function(data) {
            // console.log('STDOUT: ' + data)
            vue.term.write(unescape(encodeURIComponent(data)))
          }).stderr.on('data', function(data) {
            // console.log('STDERR: ' + data)
            vue.term.write(unescape(encodeURIComponent(data)))
          })
        })
      }).connect({
        host: server.instance.privateIp,
        port: 22,
        username: 'ec2-user',
        privateKey: require('fs').readFileSync(keyPath)
      })

      vue.term.on('data', function(data) {
        ptyProcess.write(data)
        // vue.stream.write(data)
      })
      */
    }
  },
  beforeDestroy () {
    this.term.destroy();
  },
  mounted: function() {
    ipcRenderer.on('resize', () => {
      this.resizeIt();
    });
    ipcRenderer.on('sshconnection.doSsh', async (e, server) => {
      if (this.rendered === false) {
        this.instance = server.instance;
        this.rendered = true;
        this.org = await this.$db.orgs.cfindOne({ _id: server.org }).exec();
        this.cloudService = await this.$db.cloudServices.cfindOne({_id: server.cloudServiceId}).exec();
        this.ssh(server);
        this.tab = server.tab;
      }
    });
  }
};
</script>
<style>
@import '../../assets/css/sshconnection.css'
</style>
