package main

import (
	"time"
	"fmt"
	"github.com/ethereum/go-ethereum/p2p"
	"math/big"
	"net"
)

const (
	BlockHeadersMsg   = 1
	GetBlockBodiesMsg = 2
	BlockBodiesMsg    = 3
	baseMsg     = uint64(16)
)

type BlockHeader struct {
	Number *big.Int
	GasLimit *big.Int
	GasUsed *big.Int
	Time *big.Int
}
type Transaction struct{
}
type Block struct {
	Header *BlockHeader
	Transactions []*Transaction
	Sig []*string
}

type P2PMessageNode struct {
	name string
	protos []p2p.Protocol
	blocks []*Block
	headers []*BlockHeader
}

func main(){
	mainP2PTest()
}

func mainP2PTest(){

	srcNode := P2PMessageNode{name: "srcNode"}

	header1 := &BlockHeader{big.NewInt(0),
	big.NewInt(100),
	big.NewInt(100),
	big.NewInt(time.Now().Unix())}

	header2 := &BlockHeader{big.NewInt(1),
		big.NewInt(100),
		big.NewInt(100),
		big.NewInt(time.Now().Unix())}

	block1 := &Block{Header: header1}
	block2 := &Block{Header: header2}
	srcNode.headers = append(srcNode.headers, header1, header2)
	srcNode.blocks = append(srcNode.blocks, block1, block2)

	destNode := P2PMessageNode{name: "destNode"}

	fmt.Printf("srcNode = %v \n", srcNode)
	fmt.Printf("destNode = %v \n", destNode)
	proto := p2p.Protocol{
		Name:   "test",
		Length: 4,
		Run: func(peer *p2p.Peer, rw p2p.MsgReadWriter) error {
			fmt.Println("proto run start")
			for {
				msg, err := rw.ReadMsg()
				fmt.Printf("msg=%v err=%v, ", msg, err)
				if err != nil {
					return err
				}
				fmt.Printf("msgcode = %d \n", msg.Code)

				switch {
				case msg.Code == BlockHeadersMsg:
					var query BlockHeader
					if err := msg.Decode(&query); err != nil {
						return nil
					}

					fmt.Printf("peer = %v recieve BlockHeadersMsg header = %v\n", peer.ID(), query)
					fmt.Printf("peer = %v send GetBlockBodiesMsg\n", peer.ID())
					p2p.Send(rw, GetBlockBodiesMsg, query)

				case msg.Code == GetBlockBodiesMsg:
					var query BlockHeader
					if err := msg.Decode(&query); err != nil {
						return nil
					}
					fmt.Printf("peer = %v recieve GetBlockBodiesMsg\n header.number = %d \n", peer.ID(), query.Number)
					fmt.Printf("peer = %v send BlockBodiesMsg\n", peer.ID())
					var block *Block
					for _, b := range srcNode.blocks{
						if cmp := b.Header.Number.Cmp(query.Number); cmp == 0 {
							block = b
						}
					}
					p2p.Send(rw, BlockBodiesMsg, block)

				case msg.Code == BlockBodiesMsg:
					var query Block
					if err := msg.Decode(&query); err != nil {
						return nil
					}
					fmt.Printf("peer = %v recieve BlockBodiesMsg block = %v\n", peer.ID(), query)
				default:
					//return fmt.Printf(ErrInvalidMsgCode, "%v", msg.Code)
				}
			}
			return nil

		},
	}

	fdSrc, fdDest := net.Pipe()

	srcCloser, fdSrcCon, _ := p2p.TestNewPeer(fdSrc, []p2p.Protocol{proto})
	defer srcCloser()

	destCloser, _, _ := p2p.TestNewPeer(fdDest, []p2p.Protocol{proto})
	defer destCloser()

	p2p.Send(fdSrcCon, baseMsg + BlockHeadersMsg, srcNode.headers[0])


	select {
	case <-time.After(5 * time.Second):
		fmt.Println(" timeout")
	}
}
