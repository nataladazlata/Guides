# Guides HERMES

## Prerequisites:

- install the two necessary nodes and fully synchronize them
- since the relay process must be able to request the network back in height for at least 2/3 of the unbonding period (trusting period = 2/3 of the unbonding period), it is recommended to use pruning settings that will keep the full state of the chain for a longer period of time than unbonding period
- if the nodes are not on a server with a relay, then you need to open RPC ports for Hermes
- indexing on nodes should be set to kv
- you need to create separate wallets for relaying and replenish them
- install Hermes

```
mkdir -p $HOME/.hermes/bin
wget https://github.com/informalsystems/hermes/releases/download/v1.0.0/hermes-v1.0.0-x86_64-unknown-linux-gnu.tar.gz && \
tar -C $HOME/.hermes/bin/ -vxzf hermes-v1.0.0-x86_64-unknown-linux-gnu.tar.gz && \
echo 'export PATH="$HOME/.hermes/bin:$PATH"' >> $HOME/.bash_profile && \
source $HOME/.bash_profile
```

## Config
```
sudo tee $HOME/.hermes/config.toml > /dev/null <<EOF
[global]
log_level = 'info'

[mode]

[mode.clients]
enabled = true
refresh = true
misbehaviour = false

[mode.connections]
enabled = false

[mode.channels]
enabled = false

[mode.packets]
enabled = true
clear_interval = 100
clear_on_start = true
tx_confirmation = true

[rest]
enabled = true
host = '0.0.0.0'
port = 3000

[telemetry]
enabled = true
host = '0.0.0.0'
port = 3001

[[chains]]
### CHAIN_1 ###
id = 'osmosis-1'
rpc_addr = 'http://127.0.0.1:26657'
grpc_addr = 'http://127.0.0.1:9090'
websocket_addr = 'ws://127.0.0.1:26657/websocket'
rpc_timeout = '10s'
account_prefix = 'osmo'
key_name = 'wallet'
address_type = { derivation = 'cosmos' }
store_prefix = 'ibc'
max_gas = 15000000
gas_price = { price = 0.0027, denom = 'uosmo' }
gas_multiplier = 1.1
max_msg_num = 15
max_tx_size = 180000
clock_drift = '5s'
max_block_time = '30s'
trusting_period = '<enter_your_value>'
trust_threshold = { numerator = '1', denominator = '3' }
memo_prefix = '<NAME_Relayer>'
[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-253'], #L1
]

### CHAIN_2 ###
id = 'genesis_29-2'
rpc_addr = 'http://127.0.0.1:36657'
grpc_addr = 'http://127.0.0.1:9190'
websocket_addr = 'ws://127.0.0.1:36657/websocket'
rpc_timeout = '10s'
account_prefix = 'genesis'
key_name = 'wallet'
address_type = { derivation = 'ethermint', proto_type = { pk_type = '/ethermint.crypto.v1.ethsecp256k1.PubKey' } }
store_prefix = 'ibc'
max_gas = 15000000
gas_price = { price = 20000000000, denom = 'el1' }
gas_multiplier = 1.1
max_msg_num = 15
max_tx_size = 180000
clock_drift = '5s'
trusting_period = '<enter_your_value>'
trust_threshold = { numerator = '1', denominator = '3' }
memo_prefix = '<NAME_Relayer>'
[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-1'], #OSMOSIS
]

EOF
```

## Check Config
```
hermes config validate

hermes health-check
```

## Add wallets
```
# creat files
sudo tee $HOME/.hermes/osmosis-1.mnemonic > /dev/null <<EOF
<ENTER_YOU_MNEMONIC>
EOF

sudo tee $HOME/.hermes/genesis_29-2 > /dev/null <<EOF
<ENTER_YOU_MNEMONIC>
EOF

# add wallet
hermes keys add --chain osmosis-1 --mnemonic-file $HOME/.hermes/osmosis-1.mnemonic
hermes keys add --chain genesis_29-2 --mnemonic-file $HOME/.hermes/genesis_29-2.mnemonic

# see wallets
hermes keys list --chain osmosis-1
hermes keys list --chain genesis_29-2

# see balance wallets
hermes keys balance --chain osmosis-1
hermes keys balance --chain genesis_29-2
```

## Create servise
```
sudo tee /etc/systemd/system/hermesd.service > /dev/null <<EOF
[Unit]
Description=hermes
After=network-online.target

[Service]
User=$USER
ExecStart=$(which hermes) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable hermesd
systemctl restart hermesd && journalctl -u hermesd -f -o cat
```


