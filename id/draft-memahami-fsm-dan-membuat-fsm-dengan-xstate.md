
## Apa itu Finite State Machine?

Apa itu finite state machine? Cara saya memahami istilah ini adalah dengan mencoba membedah dan memahami tiap kata yang ada dalam istilah ini:

- Machine
    
    - Machine di sini merujuk kepada “a model of a system”. Model sendiri bisa diartikan sebagai sebuah representasi yang informatif dari sesuatu (yaa intinya representasi dari sistem).
        

- State
    
    - State di sini merujuk kepada informasi. Lebih spesifik lagi, informasi yang dimaksud adalah _behavior_ (perilaku atau tingkah laku) dari sebuah system.
        

Jika dua kata ini digabungkan, “state machine” dapat dipahami sebagai representasi dari _behavior_ sebuah system. Sebuah state machine, tentunya terdiri dari daftar tingkah lakunya. State machine juga mendeskripsikan transisi dari satu state ke state lain. Transisi ini dipicu oleh _input_ yang diberikan kepada state machine tersebut.

Kata terakhir, “finite”, memiliki makna bahwa state yang ada di dalam state machine jumlahnya terbatas.

## Memodelkan finite state machine kipas angin

Untuk kemudahan development, saya menggunakan extension Stately di VSCode. Extension ini sangat memudahkan dalam memodelkan FSM karena menggunakan GUI.

Kembali ke kasus kita, kita ingin memodelkan bagaimana sebuah kipas angin bekerja. Kita bisa mulai dengan pertanyaan: “apa state dari sebuah kipas angin”. Mudah, `stop` dan `spin`. `stop` artinya kipas angin kita tidak berputar, sedangkan `spin` artinya berputar. Kita bisa menuliskan object FSM ini dengan XState sebagai berikut:

```
import { createMachine } from "xstate";

export const fanMachine = createMachine({
  id: "fan",

  states: {
    stop: {},
    spin: {},
  },

  initial: "off",
});
```

Property `states` adalah daftar `state`s yang dimiliki oleh machine kita dan property `initial` menentukan `state` dari machine kita saat ia pertama kali dijalankan.

Tapi, kita belum menentukan bagaimana machine kita bisa berpindah dari satu state `stop` ke state `spin`.

### Typescript support (you can skip this part if you’re not using TypeScript)

Jika kamu juga menggunakan TypeScript, XState menyediakan types generator (typegen) dari machine kita. Berdasarkan dokumentasi ini, [https://xstate.js.org/docs/guides/typescript.html#typegen](https://xstate.js.org/docs/guides/typescript.html#typegen) , pengguna VSCode hanya perlu menginstall sebuah extension.

Lalu di dalam definisi state machine kita, kita hanya perlu menambahkan property `tsTypes: {}`. Ketika kita menyimpan file tersebut, typings dari state machine kita akan otomatis terbuat.

### State transition

Dalam XState, transisi state dipicu oleh sebuah `EVENT`. `EVENT` di sini sepadan dengan “input” yang kita singgung pada bagian pengertian FSM di atas.

Untuk kasus kita, transisi yang kita butuhkan untuk berpindah adalah dari state `stop` ke `spin` dan sebaliknya dipicu oleh user yang menyalakan atau mematikan kipas angin. Dalam XState, kita bisa menulis transisi ini pada property `on` dari sebuah state:

```
export const fanMachine = createMachine({
  id: "fan",

  states: {
    stop: {
      on: {
        "USER.PRESS.ON": "spin",
      },
    },

    spin: {
      on: {
        "USER.PRESS.OFF": "stop",
      },
    },
  },

  initial: "stop",
});
```

Misal, transisi pada state `stop` bisa dibaca:

“On `USER.PRESS.ON`, transition to `spin`”.

Yang menarik dari XState, kamu bisa menggunakan nilai string apa saja sebagai nama nilai `EVENT`. Di sini saya menggunakan convention nama aksi yang ditulis dalam uppercase dan tiap kata dipisahkan oleh titik alih-alih oleh spasi.

Untuk membuat generated typings yang lebih baik, kita bisa menuliskan `EVENT` apa saja yang dikenali oleh machine kita. Misalnya:

```
type MachineEvent = { type: "USER.PRESS.ON" } | { type: "USER.PRESS.OFF" };

export const fanMachine = createMachine({
  id: "fan",
  tsTypes: {} as import("./fanMachine.fsm.typegen").Typegen0,
  schema: {
    events: {} as MachineEvent,
  },
  // ...
});
```

### Memperkaya state machine dengan informasi tambahan

Kita sudah berhasil membuat kipas angin kita menyala dan mati. Tapi, bagaimana dengan kecepatan putaran kipas? Di XState, informasi tambahan (atau sederhananya, data) yang diketahui oleh mesin disimpan ke dalam `context`.

Dalam kasus kita, informasi tambahan yang dibutuhkan adalah kecepatan dari kipas.

Kita bisa menambahkan kecepatan ke dalam property `context`:

```
export const fanMachine = createMachine({
  id: "fan",
  context: {
    fanSpeed: 0,
  },
  // ...
});
```

Kita juga bisa membuat type untuk context dan menambahkannya ke dalam schema:

```
type MachineContext = {
  fanSpeed: number;
};

// ...

export const fanMachine = createMachine({
  id: "fan",
  schema: {
    events: {} as MachineEvent,
    context: {} as MachineContext,
  },
  // ...
});
```

### Mengubah nilai dari `context` dengan `action`

Oke, kita sudah menambahkan kecepatan kipas, `fanSpeed` , ke dalam machine melalui `context`. Tapi, ketika `state` transisi dari `stop` ke `spin`, `fanSpeed` dari kipas angin kita masih 0!

Untuk merubah nilai dari `context`, kita bisa memanfaatkan salah satu fitur dari XState, yaitu `actions`.

Di XState, actions adalah salah satu bentuk side-effect yang bisa di-trigger. actions bisa dipicu oleh `EVENT` atau ketika machine masuk atau keluar dari sebuah `state`. action dalam XState adalah pure function; umumnya bersifat _synchronous_. Kita bisa menulis fungsi untuk merubah `context` dengan menggunakan `action`.

Pertama, mari kita update `EVENT` yang kita kirim ketika transisi dari `stop` ke `spin` dan sebaliknya untuk memicu `action` yang akan merubah nilai dari `fanSpeed`. Mari kita beri nama `action` ini `changeFanSpeed`. `action` ini ditambahkan pada property `actions` yang ada di dalam `EVENT`.

```
export const fanMachine = createMachine({
  id: "fan",
  // ...
  states: {
    stop: {
      on: {
        "USER.PRESS.ON": {
          target: "spin",
          // v let's add action here!
          actions: "changeFanSpeed",
        },
      },
    },

    spin: {
      on: {
        "USER.PRESS.OFF": {
          target: "stop",
          // v let's add action here!
          actions: "changeFanSpeed",
        },
      },
    },
  },
  // ...
});
```

Langkah selanjutnya adalah menuliskan implementasi dari action `changeFanSpeed`.

Berdasarkan [dokumentasinya](https://www.instagram.com/reel/Cz_x_YioA3C/?igshid=NTc4MTIwNjQ2YQ==), fungsi `createMachine` menerima 2 argumen, yang pertama adalah konfigurasi machine dan yang kedua adalah `options`. Salah satu property dari `options` adalah `actions`, tempat di mana kita menuliskan implementasi dari actions.