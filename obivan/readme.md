
Flash:

'''bash
 ~/Stažené/esptool-linux-amd64/esptool -p /dev/ttyACM0 -b 460800 --before default-reset --after hard-reset --chip esp32s3 write-flash --flash-mode dio --flash-size detect --flash-freq 40m 0x0 .esphome/build/obivan-s7/.pioenvs/obivan-s7/bootloader.bin 0x8000 .esphome/build/obivan-s7/.pioenvs/obivan-s7/partitions.bin 0x10000 .esphome/build/obivan-s7/.pioenvs/obivan-s7/firmware.bin
'''

Compile:

docker exec flamboyant_wilbur esphome compile obivan-s7-full.yaml | cat

Validate:

