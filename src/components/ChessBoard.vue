<template>
    <div class="chessbox">
        <table>
            <tr v-for="(row, ypart) in chessboard" v-bind:key="ypart">
                <td class="chessplace" :class="{selected: isSelected(xpart, ypart), possible: isPossible(xpart, ypart), heat: isUnderHeat(xpart, ypart)}" v-for="(place, xpart) in row" v-bind:key="xpart" v-html="pieceCode(place)" @click="doClick(place, xpart, ypart)">
                </td>
            </tr>
        </table>
        <div class="gamebox">{{onMove}} is on move.</div>
    </div>
</template>

<script>
    import _ from 'lodash';

    //Needs en passant

    export default {
        name: "ChessBoard",
        data() {
            return{
                chessboard: [],
                chesspieces: {
                    white: {
                        pawn: "&#9817;",
                        rook: "&#9814;",
                        knight:"&#9816;",
                        bishop:"&#9815;",
                        queen:"&#9813;",
                        king:"&#9812;"
                    },
                    black: {
                        pawn: "&#9823;",
                        rook: "&#9820;",
                        knight:"&#9822;",
                        bishop:"&#9821;",
                        queen:"&#9819;",
                        king:"&#9818;"
                    }
                },
                selectedItem: {
                    x: 0,
                    y: 0,
                    piece: {}
                },
                possibleMoves: [
                    {
                        x: 0,
                        y: 0
                    }
                ],
                heatmap: [],
                onMove: 'white'
            }
        },
        mounted() {
            this.clearBoard();
            this.resetBoard();
        },
        methods: {
            doClick(place, xpart, ypart){
                console.log("click")
                console.log("for " + place.piece)

                //If its a possible move
                if (this.isPossible(xpart, ypart) && this.onMove === 'white') {
                    console.log("doing move for " + this.selectedItem.piece.piece)

                    //get old x and y
                    let oldx = this.selectedItem.x;
                    let oldy = this.selectedItem.y;

                    //move player
                    this.doMove(this.chessboard, oldx, oldy, xpart, ypart)

                    //set selection to 0, possible moves to 0 and black on move
                    this.selectItem({}, -1, -1)
                    this.possibleMoves = [];
                    this.onMove = 'black';

                    this.aiMove();

                    return
                }

                //Select the item
                this.selectItem(place, xpart, ypart)
                if (place.color === 'black') return;

                this.possibleMoves = this.checkMoves(this.chessboard, place, xpart, ypart)
                this.heatmap = this.createHeatMap(this.chessboard, (place.color === 'white') ? 'black' : 'white');

                //check moves for mate and remove some if so
                this.possibleMoves = this.checkMovesforCheck(this.possibleMoves, this.chessboard, place, xpart, ypart)
            },
            checkMovesforCheck(moves, board, place, xpart, ypart){

                //big oof. check if the move doesnt put the king in danger
                let movesleft = [];

                for (let ii in moves){

                    console.log('[mate check]checking move ' + ii)

                    //copy the board
                    let boardCopy = JSON.parse(JSON.stringify(this.chessboard));

                    //do the move
                    this.doMove(boardCopy, xpart, ypart, moves[ii].x,moves[ii].y)

                    //get heatmap
                    let heatmap = this.createHeatMap(boardCopy, (place.color === 'white') ? 'black' : 'white');

                    //king still under fire?
                    let king = this.getItem(boardCopy, {piece: 'king', color: place.color})
                    let kingUnderHeat = this.positionContained(king, heatmap)

                    //if not alleviated
                    if (!kingUnderHeat){
                        console.log('added possible move')
                        movesleft.push(moves[ii])
                    }
                }

                return movesleft

            },
            aiMove(){
                let allBlackItems = this.getItems(this.chessboard, {color: 'black'})
                console.log(allBlackItems)
                return allBlackItems;
            },
            getItems(board, obj){
                let items = [];
                _.forEach(board, (row, xx) =>{
                    _.forEach(_.filter(row, obj), (item, yy) => items.push({
                        piece: item.piece,
                        color: item.color,
                        x: xx,
                        y: yy
                    }))
                })
                return items;
            },
            doMove(board, oldx, oldy, xpart, ypart){

                board[ypart][xpart] = board[oldy][oldx];
                board[oldy][oldx] = {};

                //pawn becomes queen
                let piece = board[ypart][xpart];

                if (piece.piece === 'pawn'){
                    console.log('ispawn')
                    if (piece.color === 'white' && ypart === 0){
                        console.log('is white and y is 0')
                        board[ypart][xpart].piece = 'queen'
                    }
                    if (piece.color === 'black' && ypart === 7){
                        board[ypart][xpart].piece = 'queen'
                    }
                }
            },
            selectItem(piece, xpart, ypart){
                this.selectedItem = {
                    x: xpart,
                    y: ypart,
                    piece: piece
                };
            },
            createHeatMap(board, color){
                let heatmap = [];
                for (let xx = 0; xx < 8; xx++) {
                    for (let yy = 0; yy < 8; yy++) {
                        let place = board[yy][xx]
                        if (place.piece && place.color === color){
                            heatmap = heatmap.concat(this.checkMoves(board, place, xx, yy, true))
                        }
                    }
                }
                return heatmap;
            },
            checkMoves(board, piece, xpart, ypart, isForHeatMap){
                console.log('checking moves');
                let moves = [];

                if (piece.piece === 'pawn'){
                    console.log('checking for pawn');

                    //way
                    let way = (piece.color === 'white') ? -1 : 1

                    //1 up
                    if (!isForHeatMap){
                        if (!this.spotIsTaken(board, xpart, ypart + way)){
                            moves.push({x: xpart, y: ypart + way})
                            //first row
                            if (ypart === 6 || ypart === 1){
                                if (!this.spotIsTaken(board, xpart, ypart + (way*2))) moves.push({x: xpart, y: ypart + (way*2)})
                            }
                        }
                    }

                    //heat map
                    if (isForHeatMap){
                        moves.push({x: xpart - 1, y: ypart + way})
                        moves.push({x: xpart + 1, y: ypart + way})
                    }

                    //sideways
                    if (this.canTakeEnemy(board, xpart-1, ypart + way, piece.color)){moves.push({x: xpart - 1, y: ypart + way})}
                    if (this.canTakeEnemy(board, xpart+1, ypart + way, piece.color)){moves.push({x: xpart + 1, y: ypart + way})}
                }

                if (piece.piece === 'queen'){
                    console.log('checking for queen');

                    let directions = [
                        [-1, -1], [0, -1], [1, -1],[-1, 0],
                        [1, 0], [-1, 1], [0, 1], [1, 1]
                    ]

                    for (let i = 0; i < 8; i ++) {
                        let moves_tmp = this.checkLine(board, piece.color, xpart, ypart, directions[i])
                        moves = moves.concat(moves_tmp);
                    }
                }

                if (piece.piece === 'rook'){
                    console.log('checking for rook');

                    let directions = [
                         [0, -1], [-1, 0],
                        [1, 0], [0, 1],
                    ]

                    for (let i = 0; i < 4; i ++) {
                        let moves_tmp = this.checkLine(board, piece.color, xpart, ypart, directions[i])
                        moves = moves.concat(moves_tmp);
                    }
                }

                if (piece.piece === 'bishop'){
                    console.log('checking for bishop');
                    let directions = [
                        [-1, -1], [1, -1],
                        [-1, 1], [1, 1],
                    ]
                    for (let i = 0; i < 4; i ++) {
                        let moves_tmp = this.checkLine(board, piece.color, xpart, ypart, directions[i])
                        moves = moves.concat(moves_tmp);
                    }
                }

                if (piece.piece === 'knight'){
                    console.log('checking for knight');
                    let directions = [
                        [2, -1], [2, 1], // right
                        [-1, -2], [1, -2], // top
                        [-2, -1], [-2, 1], // left
                        [-1, 2], [1, 2], // down
                    ]
                    for (let i = 0; i < 8; i ++){
                        let xcheck = xpart + (directions[i][0]); let ycheck = ypart + (directions[i][1]);

                        if (!this.checkSpaceDoesntExists(xcheck, ycheck)){
                            if (this.canTakeEnemy(board, xcheck, ycheck, piece.color) || !this.spotIsTaken(board, xcheck, ycheck) ){
                                moves.push({x: xcheck, y: ycheck})
                            }
                        }
                    }
                }
                if (piece.piece === 'king'){
                    console.log('checking for king');

                    let directions = [
                        [-1, -1], [0, -1], [1, -1],
                        [-1, 0], [1, 0],
                        [-1, 1], [0, 1], [1, 1],
                    ]

                    for (let i = 0; i < 8; i ++){
                        let xcheck = xpart + (directions[i][0]); let ycheck = ypart + (directions[i][1]);
                        if (!this.checkSpaceDoesntExists(xcheck, ycheck)){
                            if (this.canTakeEnemy(board, xcheck, ycheck, piece.color) || !this.spotIsTaken(board, xcheck, ycheck) ){
                                moves.push({x: xcheck, y: ycheck})
                            }
                        }
                    }
                }
                return moves;
            },
            checkLine(board, color, xpart, ypart, moveType){
                let moves = [];

                //horizontal checker
                for (let i = 1; i < 7; i ++){
                    let xcheck = xpart + (moveType[0] * i); let ycheck = ypart + (moveType[1] * i);

                    if (!this.checkSpaceDoesntExists(xcheck, ycheck)){
                        let canTakeEnemy = this.canTakeEnemy(board, xcheck, ycheck, color);
                        let spotIsTaken = this.spotIsTaken(board, xcheck, ycheck);
                        if (canTakeEnemy || !spotIsTaken ){
                            moves.push({x: xcheck, y: ycheck})
                            if (canTakeEnemy) i = 7;
                        }else{
                            i = 7;
                        }
                    }
                }

                return moves;
            },
            spotIsTaken(board, x, y){
                if (this.checkSpaceDoesntExists(x, y)) return false;
                return (board[y][x].piece !== undefined);
            },
            canTakeEnemy(board, x, y, color){
                if (this.checkSpaceDoesntExists(x, y)) return false;
                let obj = board[y][x]
                return (obj.piece !== undefined && obj.color !== color);
            },
            checkSpaceDoesntExists(x, y){
                if (x < 0) return true;
                if (y < 0) return true;
                if (x > 7) return true;
                if (y > 7) return true;
                return false;
            },
            isSelected(x, y){
                return (x === this.selectedItem.x && y === this.selectedItem.y)
            },
            isPossible(x, y){
                if (_.find(this.possibleMoves, {x: x, y: y})) return true
                return false;
            },
            isUnderHeat(x, y){
                if (_.find(this.heatmap, {x: x, y: y})) return true
                return false;
            },
            positionContained(obj, map){
                return !!_.find(map, {x: obj.x, y: obj.y});
            },
            pieceCode(place){
                return (place.color === 'white') ? this.chesspieces.white[place.piece] : this.chesspieces.black[place.piece];
            },
            clearBoard(){
                this.chessboard = [];
                for (let i = 0; i < 8; i++) {
                    let row = [];
                    this.chessboard.push(row);
                    for (let i = 0; i < 8; i++) {
                        let piece = {};
                        row.push(piece);
                    }
                }
            },
            resetBoard(){
                //White pawns
                let row = [];
                for (let i = 0; i < 8; i++) {
                    let piece = {
                        piece: 'pawn',
                        color: 'white'
                    }
                    row.push(piece);
                }
                this.chessboard[6] = row;

                this.chessboard[7] = [
                    {piece: 'rook', color: 'white'},
                    {piece: 'knight', color: 'white'},
                    {piece: 'bishop', color: 'white'},
                    {piece: 'queen', color: 'white'},
                    {piece: 'king', color: 'white'},
                    {piece: 'bishop', color: 'white'},
                    {piece: 'knight', color: 'white'},
                    {piece: 'rook', color: 'white'},
                ];

                //black pawns
                row = [];
                for (let i = 0; i < 8; i++) {
                    let piece = {
                        piece: 'pawn',
                        color: 'black'
                    }
                    row.push(piece);
                }
                this.chessboard[1] = row;
                //
                this.chessboard[0] = [
                    {piece: 'rook', color: 'black'},
                    {piece: 'knight', color: 'black'},
                    {piece: 'bishop', color: 'black'},
                    {piece: 'queen', color: 'black'},
                    {piece: 'king', color: 'black'},
                    {piece: 'bishop', color: 'black'},
                    {piece: 'knight', color: 'black'},
                    {piece: 'rook', color: 'black'},
                ];
            }
        }
    }
</script>

<style scoped>
    div.chessbox{
        display: flex;
        flex-direction: row;
    }

    .gamebox{
        margin: 10px;
        padding: 10px;
        width: 100%;
        border: 7px solid #000000;
        border-radius: 8px;
    }

    table {
        min-width: 620px;
        min-height: 620px;
        margin: 0 auto;
        border-collapse: collapse;
        border: 20px solid black;
    }

    .chessplace{
        width: 70px;
        height: 70px;
        background: #a5a5a5;

        font-size:50px;
        font-weight: bold;
        text-align:center;
        display: table-cell;
        vertical-align:middle;

        cursor: pointer;
    }

    tr:nth-child(odd) td:nth-child(even), tr:nth-child(even) td:nth-child(odd) {
        background: #ffffff;
    }

    td.chessplace.selected{
        background: #ffd5d5!important;
    }

    td.chessplace.heat{
        background: #de8080 !important;
    }

    td.chessplace.possible{
        background: rgba(40, 44, 161, 0.5) !important;
    }
</style>