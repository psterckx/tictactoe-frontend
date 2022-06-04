
<template>
  <div class="flex flex-col justify-between h-screen">
    <div>
      <h1 class="my-6 text-2xl md:text-4xl flex justify-center italic">
        Tic Tac Toe
      </h1>
      <div class="flex justify-center -ml-12">
        <div
          class="rounded-full h-6 w-6 mr-6 mt-4 self-start"
          :class="{
            'bg-orange-500 animate-pulse': isConnection('CONNECTING'),
            'bg-green-500': isConnection('ONLINE'),
            'bg-gray-500': isConnection('OFFLINE'),
            'animate-pulse':
              isConnection(['ONLINE', 'CONNECTING']) &&
              isQueue(['WAITING_FOR_OPPONENT', 'REQUESTING_GAME']),
          }"
        ></div>
        <GameBoard
          :enabled="thisPlayersTurn"
          @markSquare="markSquare"
          :board="game.state.board"
          :marker="game.marker"
        />
      </div>
      <div class="flex justify-center mt-10 flex-wrap mx-10">
        <button
          @click="connect()"
          :class="{
            'opacity-50': isConnection('ONLINE'),
            'hover:-translate-y-1': isConnection('OFFLINE'),
          }"
          class="bg-green-300 rounded-md p-2 mb-4 mx-2 transition-transform"
          :disabled="isConnection('ONLINE')"
        >
          Start game
        </button>

        <button
          @click="disconnect()"
          :class="{
            'opacity-50': isConnection('OFFLINE'),
            'hover:-translate-y-1': isConnection('ONLINE'),
          }"
          class="bg-red-300 rounded-md p-2 mb-4 mx-2 transition-transform"
          :disabled="isConnection('OFFLINE')"
        >
          Leave game
        </button>
        <button
          @click="requestGame()"
          :class="{
            'opacity-50': !(isConnection('ONLINE') && isGameOver),
            'hover:-translate-y-1': isConnection('ONLINE') && isGameOver,
          }"
          class="bg-blue-300 rounded-md p-2 mb-4 mx-2 transition-transform"
          :disabled="!(isGameOver || opponentDisconnected)"
        >
          Play again
        </button>
      </div>
      <div class="flex flex-col justify-start space-y-1 h-28">
        <div class="flex justify-center" v-show="game.gameId">
          You are {{ game.marker }}.
        </div>
        <div class="flex justify-center" v-if="game.gameId && !isGameOver">
          {{ turnMessage }}
        </div>
        <div class="flex justify-center" v-show="opponentDisconnected">
          Your opponent has left the game.
        </div>
        <div class="flex justify-center" v-if="isGameOver">
          {{ gameOverMessage }}
        </div>
        <div
          class="flex justify-center"
          v-if="isConnection('ONLINE') && isQueue('REQUESTING_GAME')"
        >
          Requesting game...
        </div>
        <div class="flex justify-center" v-if="isQueue('WAITING_FOR_OPPONENT')">
          Searching for an opponent...
        </div>
        <div class="flex justify-center" v-if="isConnection('CONNECTING')">
          Connecting...
        </div>
      </div>
    </div>
    <footer
      class="
        flex flex-col
        justify-center
        align-middle
        text-center
        bg-gray-300
        text-gray-700
        py-6
        space-y-1
      "
    >
      <div>
        Built with VueJS and AWS by
        <a
          class="font-medium hover:text-gray-900"
          href="https://petersterckx.com"
          target="_blank"
          >Peter Sterckx</a
        >
      </div>
      <a
        href="https://gist.github.com/psterckx/b40040d5abe9e218ebd6849527389968"
        class="flex text-center justify-center space-x-2 hover:text-gray-900"
      >
        <GitHubIcon />
        <div>Source code</div>
      </a>
    </footer>
  </div>
</template>

<script lang="ts">
import GameBoard from "./components/GameBoard.vue";
import GitHubIcon from "./components/GitHubIcon.vue";

enum ConnectionState {
  CONNECTING = "CONNECTING",
  ONLINE = "ONLINE",
  OFFLINE = "OFFLINE",
}

enum QueueState {
  REQUESTING_GAME = "REQUESTING_GAME",
  WAITING_FOR_OPPONENT = "WAITING_FOR_OPPONENT",
  PLAYING = "PLAYING",
}

export default {
  name: "App",
  components: {
    GameBoard,
    GitHubIcon,
  },
  data(): {
    wsConnection?: WebSocket;
    connectionState: ConnectionState;
    queueState: QueueState;
    opponentDisconnected: boolean;
    game: {
      gameId: string;
      player: string;
      marker: string;
      state: {
        gameOver: boolean;
        whosTurn: string;
        winner: string | null;
        board: string[][];
      };
    };
  } {
    return {
      connectionState: ConnectionState.OFFLINE,
      queueState: QueueState.REQUESTING_GAME,
      opponentDisconnected: false,
      game: {
        gameId: "",
        player: "",
        marker: "",
        state: {
          gameOver: false,
          whosTurn: "",
          winner: null,
          board: [
            ["", "", ""],
            ["", "", ""],
            ["", "", ""],
          ],
        },
      },
    };
  },
  methods: {
    connect(): void {
      this.wsConnection = new WebSocket(
        "wss://e4g34uzgr5.execute-api.us-east-1.amazonaws.com/dev"
      );

      this.connectionState = ConnectionState.CONNECTING;
      this.wsConnection.onopen = () => {
        this.connectionState = ConnectionState.ONLINE;
        this.requestGame();
      };
      this.wsConnection.onclose = () => {
        this.connectionState = ConnectionState.OFFLINE;
        this.queueState = QueueState.REQUESTING_GAME;
        this.opponentDisconnected = false;
        this.resetGame();
      };
      this.wsConnection.onmessage = (event: MessageEvent<string>) => {
        const payload: { event: string; [key: string]: any } = JSON.parse(
          event.data
        );

        switch (payload.event) {
          case "WAITING_FOR_GAME":
            this.queueState = QueueState.WAITING_FOR_OPPONENT;
            break;
          case "START_GAME":
            this.queueState = QueueState.PLAYING;
            this.game.gameId = payload.gameId;
            this.game.player = payload.player;
            this.game.marker = payload.marker;
            this.game.state = payload.state;
            break;
          case "OPPONENT_DISCONNECTED":
            this.opponentDisconnected = true;
            this.game.state = payload.state;
            break;
          case "WIN":
          case "LOSE":
          case "DRAW":
          case "GAME_UPDATED":
            this.game.state = payload.state;
        }
      };
    },
    disconnect(): void {
      if (!this.wsConnection) return;
      this.wsConnection.close();
    },
    requestGame(): void {
      this.queueState = QueueState.REQUESTING_GAME;
      this.opponentDisconnected = false;
      this.resetGame();
      if (!this.wsConnection) return;
      this.wsConnection.send(JSON.stringify({ action: "requestGame" }));
    },
    markSquare(square: number[]): void {
      if (!this.wsConnection) return;

      const gameId = this.game.gameId;
      this.wsConnection.send(
        JSON.stringify({ action: "markSquare", gameId, square })
      );
    },
    isConnection(state: string | string[]): boolean {
      if (typeof state === "string") {
        return this.connectionState === state;
      }
      return state.includes(this.connectionState);
    },
    isQueue(state: string | string[]): boolean {
      if (typeof state === "string") {
        return this.queueState === state;
      }
      return state.includes(this.queueState);
    },
    resetGame() {
      this.game = {
        gameId: "",
        player: "",
        marker: "",
        state: {
          gameOver: false,
          whosTurn: "",
          winner: null,
          board: [
            ["", "", ""],
            ["", "", ""],
            ["", "", ""],
          ],
        },
      };
    },
  },
  computed: {
    thisPlayersTurn(): boolean {
      if (!this.game.player || this.game.state.gameOver) return false;
      return this.game.state.whosTurn === this.game.player;
    },
    gameOverMessage(): string {
      if (!this.isGameOver) return "";
      if (this.isWin) {
        return "You won!";
      } else if (this.isDraw) {
        return "It's a draw!";
      }
      return "You lost!";
    },
    turnMessage(): string {
      return this.thisPlayersTurn
        ? "It's your turn."
        : "Your opponent is playing...";
    },
    isGameOver(): boolean {
      return this.game.state.gameOver;
    },
    isDraw(): boolean {
      return this.game.state.winner === "draw";
    },
    isWin(): boolean {
      return Boolean(
        this.game.state.winner && this.game.state.winner === this.game.player
      );
    },
    isLose(): boolean {
      return Boolean(
        this.game.state.winner && this.game.state.winner !== this.game.player
      );
    },
  },
};
</script>