# Please change on line of code in /fabric/fabric-samples/test-network/scripts
# Filename : createChannel.sh
# Line Number: 40

# From: osnadmin channel join --channel-id $CHANNEL_NAME

# Changes on : --channel-id To --channelID

* `osnadmin channel join --channelID $CHANNEL_NAME --config-block ./channel-artifacts/${CHANNEL_NAME}.block -o localhost:7053 --ca-file "$ORDERER_CA" --client-cert "$ORDERER_ADMIN_TLS_SIGN_CERT" --client-key "$ORDERER_ADMIN_TLS_PRIVATE_KEY" >&log.txt`
