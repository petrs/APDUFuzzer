# Installation

## Pip install

```
$> pip install apdu-fuzzer

$> apdu-fuzz --help
usage: apdu-fuzz [-h] [--start_ins START_INS] [--end_ins END_INS]
                 [--output OUTPUT_FILE] [--no-trust]

Fuzz smartcard api.

optional arguments:
  -h, --help            show this help message and exit
  --start_ins START_INS
                        Instruction to start fuzzing at
  --end_ins END_INS     Instruction to stop fuzzing at
  --output OUTPUT_FILE  File to output results to
  --no-trust

$> apdu-afl-fuzz --help
```

## Installation on Debian based Linux
For the fuzzer to work we need https://github.com/mit-ll/LL-Smartcard and its dependencies:

```
git clone https://github.com/mit-ll/LL-Smartcard
cd LL-Smartcard
./install_dependencies.sh
python2 setup.py install
```

## Installation on MacOS

For the fuzzer to work we need https://github.com/mit-ll/LL-Smartcard and its dependencies:

```
brew install swig
brew install pcsc-lite
pip install llsmartcard-ph4
```

## Experimental installation with pip

```
# Create virtual environment
python -m venv --upgrade venv
cd python

# Install all project dependencies
../venv/bin/pip install --find-links=. --no-cache .

# Install AFL deps (cython required)
# Mac:
brew install afl-fuzz

# Others:
cd /tmp
wget http://lcamtuf.coredump.cx/afl/releases/afl-latest.tgz
tar -xzvf afl-latest.tgz
cd afl-*
make
sudo make install

# Install python dependencies
../venv/bin/pip install --find-links=. --no-cache .[afl]
```

## AFL fuzzing

```
AFL <-> Client <-> Server <-> Card

+----------------------------------+
|  AFL                             |
|  | |                             |                                   +-------------------+
|  | |          +----------------+ |         +------------------+      |                   |
|  | |   stdin  |                | |  socket |                  |      | +---+             |
|  | +----------|     Client     |------------      Server      -------- |-|-|    Card     |
|  |            |                | |         |                  |      | +---+             |
|  | +------+   +--------|-------+ |         +------------------+      |                   |
|  +-| SHM  |------------+         |                                   +-------------------+
|    +------+                      |
|                                  |
+----------------------------------+
```

(ascii by https://textik.com/)

Notes:

- Server is started first, connects to the card and listens on the socket for raw data to send to the card.
Does not process input data in any way.

- Server stores raw responses from the card to the data files.

- Server is able to reconnect to the card if something goes wrong.

- Client is started by AFL. AFL sends input data via STDIN, forking the client with each new fuzz input.
PCSC does not like forking with AFL this server/client architecture was required.

- Client is forked by the AFL after python is initialized. Socket can be opened either before fork or after the fork.
After fork is safer as each fuzz input has a new connection but a bit slower. Opening socket before fork also works
but special care needs to be done on broken pipe exception - reconnect logic is needed. This is not implemented now.

- Client post-processes input data generated by the AFL, e.g., generates length fields, can do TLV, etc.


Communication between server/client:

- Client sends `[0, buffer]`. Buffer is raw data structure to be sent to the card. `0` is the type / status

- Server responds with: `status 1B | SW1 1B | SW2 1B | timing 2B | data 0-NB`

```
+----+----+----+--------+------------------------+
|    |    |    |        |                        |
| 0  | SW | SW | timing |     response data      |
|    |  1 |  2 |        |                        |
+----+----+----+--------+------------------------+
```

Client then takes response from the socket, and uses modified [python-afl-ph4] to add trace to the shared memory segment
that is later analyzed by AFL to determine whether this fuzz input lead to different execution trace than the previous one.

Currently the trace bitmap is done in the following way:

```python
afl.trace_offset(hashxx(bytes([sw1, sw2])))
afl.trace_offset(hashxx(timing))
afl.trace_offset(hashxx(bytes(data)))
```

Fowler-Noll-Vo hash function used in `afl.trace_buff` is not very good with respect to the zero buffers. The timing
was usually not affecting the bitmap so we switched to very fast hash function `hashxx` for the offset computation.

### Running

Start server sitting on the card:

```
python main_afl.py --server
```


Testing if the client works:

```
echo -n '0000' | ../venv/bin/python main_afl.py --client --output ydat.json --log ylog.txt
cat ylog.txt
```

AFL with forking & TCP communication with the server:

```
../venv/bin/py-afl-fuzz -m 500 -t 5000 -o result/ -i inputs/ -- ../venv/bin/python main_afl.py --client --output ydat.json --log ylog.txt
```

## Local development

The apdu_fuzzer package is using relative imports.
For development and debugging in the local directory is thus required to
load main package `apdu_fuzzer` first. Otherwise you get the following error:

```bash
$> ../venv/bin/python ../apdu_fuzzer/main_afl.py --help

Traceback (most recent call last):
  File "../apdu_fuzzer/main_afl.py", line 17, in <module>
    from .utils.card_interactor import CardInteractor
ModuleNotFoundError: No module named '__main__.utils'; '__main__' is not a package
```

For the local execution use the wrappers in the main directory:

```bash
$> ../venv/bin/python ../main_afl.py --help
```

OR you can use package import (limitation: relative import is not supported, so it has to be executed from the root of this repository)

```bash
$> ./venv/bin/python -m apdu_fuzzer.main_afl --help
```


[python-afl-ph4]: https://github.com/ph4r05/python-afl
