<script setup lang="ts">
import { onMounted, ref } from "vue";
import { Terminal } from "xterm";
import { FitAddon } from "xterm-addon-fit";
import {
  WASI,
  Fd,
  PreopenDirectory,
  Directory,
  File,
  strace,
} from "@bjorn3/browser_wasi_shim";
import * as wasi from "@bjorn3/browser_wasi_shim/typings/wasi_defs";
import ubuntu from "/@/out.wasm?init";
import workerjs from "/@/worker/worker.js?url";
import {
  openpty,
  ISTRIP,
  INLCR,
  IGNCR,
  ICRNL,
  IXON,
  OPOST,
  ECHO,
  ECHONL,
  ICANON,
  ISIG,
  IEXTEN,
  Termios,
  TtyServer,
} from "xterm-pty";
const count = ref(0);
const terminalDiv = ref<HTMLDivElement>();

const xterm = new Terminal();
const startWasi = () => {
  if (terminalDiv.value) {
    xterm.open(terminalDiv.value);
    //xterm.write("aaaaaaaaa");
    // let fitAddon = new FitAddon();
    // xterm.loadAddon(fitAddon);
    // fitAddon.fit();
  }

  const { master, slave } = openpty();

  let termios = slave.ioctl("TCGETS");
  const iflag =
    termios.iflag &
    ~(/*IGNBRK | BRKINT | PARMRK |*/ (ISTRIP | INLCR | IGNCR | ICRNL | IXON));
  const oflag = termios.oflag & ~OPOST;
  const lflag = termios.lflag & ~(ECHO | ECHONL | ICANON | ISIG | IEXTEN);
  //termios.cflag &= ~(CSIZE | PARENB);
  //termios.cflag |= CS8;
  slave.ioctl(
    "TCSETS",
    new Termios(iflag, oflag, termios.cflag, lflag, termios.cc)
  );
  xterm.loadAddon(master);
  const worker = new Worker(workerjs);
  new TtyServer(slave).start(worker);

  // const doUbuntu = async () => {
  //   xterm.writeln("\x1B[93mDownloading\x1B[0m");
  //   //let wasm = await WebAssembly.compileStreaming(fetch("/rust_out.wasm"));
  //   xterm.writeln("\x1B[93mInstantiating\x1B[0m");

  //   async function load_external_file(
  //     path: RequestInfo | URL,
  //     filename: string
  //   ) {
  //     return new File(await (await (await fetch(path)).blob()).arrayBuffer());
  //   }

  //   let args = ["echo", "foo"];
  //   let env = ["RUSTC_LOG=info"];
  //   let fds: Fd[] = [
  //     new XtermStdio(xterm),
  //     new XtermStdio(xterm),
  //     new XtermStdio(xterm),
  //     new PreopenDirectory("/tmp", {}),
  //     new PreopenDirectory("/sysroot", {
  //       lib: new Directory({
  //         rustlib: new Directory({
  //           "wasm32-wasi": new Directory({
  //             lib: new Directory({}),
  //           }),
  //         }),
  //       }),
  //     }),
  //     new PreopenDirectory("/", {
  //       "hello.rs": new File(
  //         new TextEncoder().encode(`fn main() { println!("Hello World!"); }`)
  //       ),
  //     }),
  //   ];

  //   let w = new WASI(args, env, fds);

  //   let inst = await ubuntu({
  //     wasi_snapshot_preview1: strace(w.wasiImport, ["fd_prestat_get"]),
  //   });
  //   xterm.writeln("\x1B[93mExecuting\x1B[0m");
  //   console.log(inst.exports);
  //   try {
  //     w.start(inst as any);
  //   } catch (e: any) {
  //     xterm.writeln(e);
  //   }
  //   xterm.writeln("\x1B[92mDone\x1B[0m");

  //   console.log(fds);
  //   console.log((fds[5] as any).dir);
  //   console.log((fds[5] as any).dir.contents["hello.rs"].data);
  // };

  // doUbuntu();
};

onMounted(startWasi);

class XtermStdio extends Fd {
  term: Terminal;
  /*:: term: Terminal*/

  constructor(term: Terminal) {
    super();
    this.term = term;
  }
  fd_write(
    view8: Uint8Array,
    iovs: wasi.Iovec[]
  ) /*: {ret: number, nwritten: number}*/ {
    let nwritten = 0;
    for (let iovec of iovs) {
      console.log(
        iovec.buf_len,
        iovec.buf_len,
        view8.slice(iovec.buf, iovec.buf + iovec.buf_len)
      );
      let buffer = view8.slice(iovec.buf, iovec.buf + iovec.buf_len);
      this.term.write(buffer);
      nwritten += iovec.buf_len;
    }
    return { ret: 0, nwritten };
  }
}
</script>

<template>
  <h1>Hello, World</h1>
  <div ref="terminalDiv" id="terminal"></div>

  <div class="card">
    <button type="button" @click="count++">count is {{ count }}</button>
    <p>
      Edit
      <code>components/HelloWorld.vue</code> to test HMR
    </p>
  </div>

  <p>
    Check out
    <a href="https://vuejs.org/guide/quick-start.html#local" target="_blank"
      >create-vue</a
    >, the official Vue + Vite starter
  </p>
  <p>
    Install
    <a href="https://github.com/johnsoncodehk/volar" target="_blank">Volar</a>
    in your IDE for a better DX
  </p>
</template>

<style module lang="scss"></style>
