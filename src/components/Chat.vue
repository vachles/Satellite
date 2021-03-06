<template>
  <!--@contextmenu="openContext"-->
  <div id="wrapper">
    <Context v-if="showContext" :x="contextCoordsX" :y="contextCoordsY" :close="closeContext" />
    <Error />
    <Database
      v-if="
        $store.state.p2pOnline ||
          $store.state.dwellerAddress !== '0x0000000000000000000000000000000000000000'
      "
    />
    <Voice v-if="$store.state.p2pOnline &&
      $store.state.dwellerAddress !== '0x0000000000000000000000000000000000000000'"
    />
    <ScreenCapture
      v-if="
        windowBound &&
          $store.state.p2pOnline &&
          $store.state.dwellerAddress !== '0x0000000000000000000000000000000000000000'
      "
    />

    <!-- 
      Whenever the account is connected starts a background task that 
      continuously fetches the balance
     -->
    <BalanceFetcher v-if="$store.state.web3connected"/>

    <Web3 v-if="!$store.state.web3connected"/>
    <Loading
      v-else-if="!$store.state.friendsLoaded ||
          !$store.state.p2pOnline ||
          $store.state.dwellerAddress === '0x0000000000000000000000000000000000000000' ||
          !$store.state.dwellerAddress ||
          $store.state.starting
      "
    />
    <div v-else>
      <Achievement v-if="false" achievement="addFriend" />
      <CreateServer v-if="showCreateServer" :close="closeCreateServer" />
      <Calling :active="$store.state.incomingCall" :callerId="$store.state.incomingCall" />
      <!--<Polling />-->

      <div :class="`columns wrapper ${$store.state.sidebarOpen ? '' : 'wrapper-closed'} ${settingsOpen ? 'settings-open' : ''}`">
        <!--Main Chat-->
        <div class="column is-one-third sidebar-wrapper" :class="{'show': $store.state.sidebarMobileOpen}" v-if="$store.state.sidebarOpen">
          <Sidebar :toggleSettings="toggleSettings" :toggleCreateServer="toggleCreateServer" :servers="servers" :loadingServers="loadingServers" />
        </div>
        <div class="column chat-wrapper">
          <Main :class="$store.state.mainRoute == 'main' ? 'show' : 'hidden'" />
          <Server v-if="$store.state.mainRoute == 'server'" />
          <Files v-if="$store.state.mainRoute == 'files'" />
          <Friends v-if="$store.state.mainRoute == 'friends' 
                         && $store.state.web3connected" 
          />
        </div>
      </div>
      <div :class="`settings ${settingsOpen ? 'settings-open-container' : ''}`" v-if="settingsOpen">
        <Settings :toggleSettings="toggleSettings" :open="settingsOpen" />
      </div>
      <div class="footer">
        <p>
          <i :class="`fas fa-heartbeat ${$store.state.p2pOnline ? 'green' : 'red'}`"></i> P2P
          <span class="spacer"></span>
          <i class="fas fa-info-circle"></i>
          {{ $store.state.status }}
          <span class="spacer"></span>
          <span v-if="$store.state.accounts">
            <i class="fab fa-ethereum"></i>
            <b>{{ $t('footer.network') }}:</b> {{ $store.state.web3Stats.nettype.name.toUpperCase() }}
            <span class="spacer"> </span>
            <i class="fas fa-hashtag"></i>
            <b>{{ $t('footer.block_number') }}:</b> {{ $store.state.web3Stats.blockNumber }}
            <span class="spacer"> </span>
            <i class="fas fa-id-badge"></i>
            <b>{{ $t('footer.account') }}:</b> {{ $store.state.accounts[0] }}
          </span>
          <span v-else>
            {{ $t('footer.connecting') }}
          </span>
        </p>
      </div>
    </div>
  </div>
</template>

<script>
import Mousetrap from 'mousetrap';
import Voice from '@/components/media/Voice';
import ScreenCapture from '@/components/media/ScreenCapture';
import Sidebar from '@/components/sidebar/sidebar/Sidebar';
import Main from '@/components/main/main/Main';
import Error from '@/components/main/popups/error/Error';
import Files from '@/components/files/files/Files';
import Friends from '@/components/friends/friends/Friends';
import Settings from '@/components/main/settings/Settings';
import Web3 from '@/components/functional/web3/Web3';
import BalanceFetcher from '@/components/functional/web3/BalanceFetcher';
import Database from '@/components/functional/database/Database';
import Loading from '@/components/common/Loading';
import Achievement from '@/components/common/Achievement';
import Calling from '@/components/main/popups/calling/Calling';
import CreateServer from '@/components/servers/create/CreateServer';
import Context from '@/components/common/context/Context';
import Polling from '@/components/functional/Polling';
import Server from '@/components/server/Server';

import IPFS from 'ipfs-core';

import config from '@/config/config';
import Registry from '@/classes/contracts/Registry.ts';
import DwellerContract from '@/classes/contracts/DwellerContract.ts';
import ServerContract from '@/classes/contracts/ServerContract.ts';

export default {
  name: 'chat',
  components: {
    Achievement,
    Polling,
    Sidebar,
    Main,
    Error,
    Files,
    Friends,
    Settings,
    Web3,
    BalanceFetcher,
    Database,
    Loading,
    Voice,
    ScreenCapture,
    Calling,
    CreateServer,
    Context,
    Server,
  },
  data() {
    return {
      msg: 'Chat',
      showCreateServer: false,
      windowBound: false,
      settingsOpen: false,
      network: '',
      account: 0x0,
      blockNumber: 0,
      showContext: false,
      contextCoordsX: 0,
      contextCoordsY: 0,
      loadingServers: false,
      servers: []
    };
  },
  computed: {
    mainRoute() {
      return this.$store.state.mainRoute;
    },
  },
  watch: {
    mainRoute() {
      this.$nextTick(() => {
        setTimeout(() => {
          this.$store.commit('setMobileSidebar', false);
        }, 0);
      });
    },
  },
  methods: {
    checkMobile() {
      const width = window.innerWidth || document.documentElement.clientWidth || document.body.clientWidth;
      if (width <= 768) {
        this.$store.commit('setSidebar', true);
      }
    },
    openContext(event) {
      event.preventDefault();
      this.contextCoordsX = event.clientX;
      this.contextCoordsY = event.clientY;
      this.showContext = true;
    },
    closeContext() {
      this.showContext = false;
    },
    closeCreateServer() {
      this.showCreateServer = false;
      this.updateServers();
    },
    toggleCreateServer() {
      this.showCreateServer = !this.showCreateServer;
    },
    toggleSettings() {
      this.settingsOpen = !this.settingsOpen;
      if (this.settingsOpen) this.$store.commit('changeRoute', 'main');
    },
    async updateServers() {
      this.loadingServers = true;
      const registry = new Registry(this.$ethereum, config.registry[config.network.chain]);
      const dwellerContractAddress = await registry.getDwellerContract(this.$store.state.activeAccount);
      const dwellerContract = new DwellerContract(this.$ethereum, dwellerContractAddress);
      const serverAddresses = await dwellerContract.getServers(this.$store.state.activeAccount);

      const fetchServers = serverAddresses.map((serverAddress) => {
        const serverContract = new ServerContract(this.$ethereum, serverAddress);
        return serverContract.get(serverAddress);
      });
      const servers = await Promise.all(fetchServers);

      this.servers = servers;
      this.loadingServers = false;
    },
  },
  async mounted() {
    Mousetrap.bind('option+s', () => {
      this.settingsOpen = !this.settingsOpen;
    });
    Mousetrap.bind('esc', () => {
      this.settingsOpen = false;
      this.$store.commit('changeRoute', 'main');
    });
    const ipfs = await IPFS.create();
    window.ipfs = ipfs;
    const checkPeer = () => {
      if (this.$WebRTC.connection) {
        this.windowBound = true;
      } else {
        setTimeout(() => {
          checkPeer();
        }, 500);
      }
    };
    checkPeer();
    this.checkMobile();
    window.addEventListener('resize', this.checkMobile, true);

    this.updateServers();
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  .hidden {
    display: none;
  }
  .show {
    display: block;
  }
  #wrapper {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    /* min-width: 900px; */
  }
  .settings {
    position: absolute;
    top: 0;
    z-index: 3;
    right: -100vw;
    left: 100vw;
    bottom: 0;
    background: #e7ebee;
  }
  .sidebar-wrapper {
    max-width: 320px;
    height: 100%;
  }
  .chat-wrapper {
    height: 100%;
  }
  .settings-open-container {
    right: 0;
    left: 0;
    z-index: 99;
  }
  .fas {
    font-size: 8pt;
  }
  .wrapper {
    height: 100%;
    position: absolute;
    top: 0;
    bottom: 0;
    right: 0;
    left: 3.8rem;
    margin: 0;
    border-left: 1px solid #e7ebee;
  }
  .wrapper-closed {
    left: 0;
  }
  .settings-open {
    left: -100vw;
    right: 100vw;
    max-width: 100vw;
  }
  .footer {
    position: absolute;
    bottom: 0;
    right: 0;
    left: 0;
    top: calc(100% - 2rem);
    padding: 0.3rem 1.75rem;
    border-top: 1px solid #e7ebee;
    background: #fff;
    font-size: 10pt;
    color: #666;
  }
  .column {
    margin: 0;
    padding: 0;
  }
  .spacer {
    width: 15px;
    height: 100%;
    display: inline-block;
  }
  .fa-moon {
    cursor: pointer;
  }
  .red {
    color: #e74c3c;
  }
  .green {
    color: #00d0a1;
  }

  @media (max-width: 768px) {
    .sidebar-wrapper {
      position: fixed;
      z-index: 10;
      top: 0;
      bottom: 0;
      left: 0;
      width: 100%;
      transition: all .3s;
      transform: translateX(-100%);
    }

    .sidebar-wrapper.show {
      transform: translateX(0);
    }
  }
</style>
