# IR_Learn
This is an IR learning component based on the RMT module


---

Implement an IR learning feature on an ESP32-family SoC using Espressif’s `espressif/ir_learn` component (ESP-IDF ≥ v5.0, supported targets: esp32, esp32s2, esp32s3, esp32c3, esp32c6, esp32h2). The feature must:

1. **Use the `ir_learn` component**  
   - Add the `espressif/ir_learn` component from the ESP Component Registry and include its header `ir_learn.h`. [[ir_learn component](https://components.espressif.com/components/espressif/ir_learn)]

2. **Configure and create the IR learn handle**  
   - Fill an `ir_learn_cfg_t` struct with:
     - `learn_count` – number of times the same command should be learned (e.g., 4).  
     - `learn_gpio` – GPIO connected to the IR receiver output.  
     - `clk_src` – `RMT_CLK_SRC_DEFAULT`.  
     - `resolution` – `1000000` (1 MHz, 1 tick = 1 µs).  
     - `task_stack` – e.g., 4096.  
     - `task_priority` – e.g., 5.  
     - `task_affinity` – core ID or -1 for no affinity.  
     - `callback` – user callback of type `ir_learn_result_cb`.  
   - Call `ir_learn_new(&config, &handle)` to start the learning process. [[IR learn API](https://docs.espressif.com/projects/esp-iot-solution/en/latest/ir/ir_learn.html#api-reference); [Create example](https://docs.espressif.com/projects/esp-iot-solution/en/latest/ir/ir_learn.html#application-examples)]

3. **Implement the callback to handle learning states**  
   - Implement `void cb(ir_learn_state_t state, uint8_t sub_step, struct ir_learn_sub_list_head *data)` and handle:
     - `IR_LEARN_STATE_READY` – log / indicate learning is ready.  
     - `IR_LEARN_STATE_STEP` – log current step and `sub_step`.  
     - `IR_LEARN_STATE_FAIL` – log failure and optionally restart learning with `ir_learn_restart(handle)`.  
     - `IR_LEARN_STATE_END` – treat as success:
       - Save the result list pointed to by `data` for later transmission (e.g., deep copy the linked list of `ir_learn_sub_list_t`).  
       - Optionally call `ir_learn_print_raw(data)` to print the raw RMT symbols.  
       - Stop learning and free internal resources with `ir_learn_stop(&handle)`.  
     - `IR_LEARN_STATE_EXIT` – log learning exit.  
   - Follow the example pattern shown in `ir_learn_auto_learn_cb` in the documentation. [[Callback example](https://docs.espressif.com/projects/esp-iot-solution/en/latest/ir/ir_learn.html#application-examples)]

4. **Understand the learned data format**  
   - Each learned command is represented as a list (`ir_learn_sub_list_head`) of `ir_learn_sub_list_t` nodes. Each node contains:
     - `uint32_t timediff` – interval from the previous packet in milliseconds.  
     - `rmt_rx_done_event_data_t symbols` – the RMT RX symbols (array pointer + count) for that sub-packet. [[Structures](https://docs.espressif.com/projects/esp-iot-solution/en/latest/ir/ir_learn.html#structures)]

5. **Optional: validate or post-process the learned data**  
   - The component already performs automatic consistency checks across repeated learns (symbol count and level time differences, configurable via `RMT_DECODE_MARGIN_US` in menuconfig). If verification fails, it will emit `IR_LEARN_STATE_FAIL`. [[IR learn description](https://docs.espressif.com/projects/esp-iot-solution/en/latest/ir/ir_learn.html)]

6. **Optional: re-learn or clean data**  
   - To learn again, you can:
     - Call `ir_learn_restart(handle)` to restart the process.  
   - To manually manage memory if you create your own `ir_learn_list_head`/`ir_learn_sub_list_head`:
     - Use `ir_learn_add_list_node`, `ir_learn_add_sub_list_node` to build lists.  
     - Use `ir_learn_clean_data` or `ir_learn_clean_sub_data` to free them. [[IR learn API](https://docs.espressif.com/projects/esp-iot-solution/en/latest/ir/ir_learn.html#api-reference)]

7. **Implement IR send-out using learned raw data**  
   - Create an RMT TX channel (`rmt_new_tx_channel`) with:
     - `clk_src = RMT_CLK_SRC_DEFAULT`  
     - `resolution_hz = IR_RESOLUTION_HZ` (matching the learn resolution)  
     - Proper `mem_block_symbols`, `trans_queue_depth`, and `gpio_num = IR_TX_GPIO_NUM`.  
   - Apply a 38 kHz carrier using `rmt_apply_carrier` (e.g., 38 kHz, duty 0.33).  
   - Create a raw IR encoder with `ir_encoder_new(&raw_encoder_cfg, &raw_encoder)` (set `.resolution` to the same resolution).  
   - For each `ir_learn_sub_list_t` node in the saved result:
     - Delay by `timediff` (converted to ticks/ms) before sending that part.  
     - Extract `rmt_symbol_word_t *rmt_symbols = sub_it->symbols.received_symbols;` and `size_t symbol_num = sub_it->symbols.num_symbols;`.  
     - Call `rmt_transmit(tx_channel, raw_encoder, rmt_symbols, symbol_num, &transmit_cfg)` then `rmt_tx_wait_all_done(tx_channel, -1)`.  
   - Finally, disable and delete the TX channel and delete the encoder. Use the example `ir_learn_test_tx_raw()` as the reference implementation. [[Send out example](https://docs.espressif.com/projects/esp-iot-solution/en/latest/ir/ir_learn.html#send-out)]

8. **Hardware assumptions**  
   - Use a 38 kHz IR receiver module (e.g., IRM-3638T) connected to `learn_gpio`.  
   - Use a 38 kHz IR LED (e.g., IR333C) on `IR_TX_GPIO_NUM`, with proper driver circuitry. [[ir_learn README](https://github.com/espressif/esp-iot-solution/blob/master/components/ir/ir_learn/README.md)]

Provide complete, buildable C code snippets for:

- Component initialization and `ir_learn_cfg_t` setup.  
- The IR learn callback implementation.  
- Logic to start learning and handle success/failure in `app_main`.  
- The IR send-out function similar to `ir_learn_test_tx_raw()` that uses stored learning results.

Do not implement protocol-specific decoding; keep everything in raw RMT symbol form, as the component is designed to store and forward raw data only.

---

You can adjust GPIO numbers, task parameters, and `learn_count` as needed for your board and application.
