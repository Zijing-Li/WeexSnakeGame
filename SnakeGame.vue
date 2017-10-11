<template>
	<div class="content">
		<div class="game-body"
			 :style="{height:gridSize*rowNumber}"
			>
			<div class="game-row" v-for="(row, rowIndex) in gridList">
				<div class="game-grid"
					 :style="{width:gridSize, height:gridSize}"
					 v-for="(grid, columnIndex) in row">
					<image class="grid-image" v-if="grid.status!=gridStatus.empty" resize="contain" :src="gridImage(grid)"></image>
				</div>
			</div>
		</div>
		<div class="footer">
			<text class="reset-button" v-if="gameStatus!=0" @click="resetGame">{{gameStatus==1?'再玩一次':'重来'}}</text>
			<div class="control-content" v-if="gameStatus==0">
				<div class="control-row flex-center">
					<image class="control-image" resize="contain" :src="directionUp" @click="turn(walkDirection.up)"></image>
				</div>
				<div class="control-row flex-space-between">
					<image class="control-image" resize="contain" :src="directionLeft" @click="turn(walkDirection.left)"></image>
					<image class="control-image" resize="contain" :src="directionRight" @click="turn(walkDirection.right)"></image>
				</div>
				<div class="control-row flex-center">
					<image class="control-image" resize="contain" :src="directionDown" @click="turn(walkDirection.down)"></image>
				</div>
			</div>
		</div>
	</div>
</template>

<script>
	const modal = weex.requireModule('modal')
	var timer = null
	export default {
		computed: {
			gridSize: function () {
				return 750.0/this.columnNumber
			}
		},
		data: {
			gameStatus: 0, // 0进行中 1赢 2输
			gridList: [],
			snakeGridList: [],
			rowNumber: 20,
			columnNumber: 30,
			gridStatus: {
			    empty: 0,
				wall: 1,
				snakeHeader: 2,
				snakeBody: 3,
				food: 4
			},
			currentDirection: 0,
			walkDirection: {
			    left: 1,
				up: 2,
				right: -1,
				down: -2
			},
			headerHighlight: false,
			foodHighlight: false,
			speedLevel: 0,
			turning: false,
			speeds: [800, 700, 600, 500, 400, 300, 200, 100, 50],
			snakeHeaderImage1: require('@/assets/img/snake_header_1.png'),
			snakeHeaderImage2: require('@/assets/img/snake_header_2.png'),
			snakeBodyImage: require('@/assets/img/snake_body.png'),
			snakeFoodImage1: require('@/assets/img/snake_food_1.png'),
			snakeFoodImage2: require('@/assets/img/snake_food_2.png'),
			wallImageLeft: require('@/assets/img/wall_left.png'),
			wallImageTop: require('@/assets/img/wall_top.png'),
			wallImageRight: require('@/assets/img/wall_right.png'),
			wallImageBottom: require('@/assets/img/wall_bottom.png'),
			directionUp: require('@/assets/img/direction_up.png'),
			directionLeft: require('@/assets/img/direction_left.png'),
			directionRight: require('@/assets/img/direction_right.png'),
			directionDown: require('@/assets/img/direction_down.png'),
		},
		methods: {
		    turn: function (direction) {
		        if (this.turning || this.currentDirection==direction || this.currentDirection==-direction) {
		            return
				}
		        this.turning = true
				this.currentDirection = direction
			},
		    gridImage: function (grid) {
				switch (grid.status) {
					case this.gridStatus.wall:
						if (grid.row==0) {
							return this.wallImageTop
						} else if (grid.row == this.rowNumber-1) {
							return this.wallImageBottom
						} else if (grid.column == 0) {
							return this.wallImageLeft
						} else if (grid.column == this.columnNumber-1) {
							return this.wallImageRight
						}
					case this.gridStatus.snakeHeader:
					    if (this.headerHighlight) {
					        return this.snakeHeaderImage1
						} else {
					        return this.snakeHeaderImage2
						}
					case this.gridStatus.snakeBody:
					    return this.snakeBodyImage
					case this.gridStatus.food:
					    if (this.foodHighlight) {
					        return this.snakeFoodImage1
						} else {
					        return this.snakeFoodImage2
						}
					default:
					    return ' '
				}
			},
			failed: function () {
				this.gameStatus = 2
				this.stopWalk()
				toast('你输了！', 3)
			},
			success: function () {
				this.gameStatus = 1
				this.stopWalk()
				toast('你赢了！', 3)
			},
			eatSuccess: function () {
				if (this.snakeGridList.length>(this.speedLevel+1)*5-4 && this.speedLevel < this.speeds.length-1) {
					this.speedLevel++
					this.stopWalk()
					this.startWalk()
				}
			},
			nextStep: function () {
		        var header = this.snakeGridList[0]
				var nextGrid
				switch (this.currentDirection) {
					case this.walkDirection.left:
					    nextGrid = this.gridList[header.row][header.column-1]
					    break
					case this.walkDirection.up:
						nextGrid = this.gridList[header.row-1][header.column]
						break
					case this.walkDirection.right:
						nextGrid = this.gridList[header.row][header.column+1]
						break
					case this.walkDirection.down:
						nextGrid = this.gridList[header.row+1][header.column]
						break
					default:
					    nextGrid = null
					    break
				}
				if (nextGrid != null) {
		            if (nextGrid.status != this.gridStatus.empty && nextGrid.status != this.gridStatus.food) {
		                this.failed()
						return
					} else if (nextGrid.status == this.gridStatus.empty) {
						var tail = this.snakeGridList.pop()
						tail.status = this.gridStatus.empty
					}
					if (nextGrid.status == this.gridStatus.food) {
						this.randomFood()
						this.eatSuccess()
					}
					header.status = this.gridStatus.snakeBody
					nextGrid.status = this.gridStatus.snakeHeader
					this.snakeGridList.unshift(nextGrid)
					this.gridList[nextGrid.row].splice(nextGrid.column, 1, nextGrid)
					if (this.turning) {
		                this.turning = false
					}
					if (nextGrid.status == this.gridStatus.food) {
						this.eatSuccess()
					}
				}
			},
			startWalk: function () {
				timer = setInterval(this.nextStep, this.speeds[this.speedLevel])
			},
			stopWalk: function () {
				clearInterval(timer)
				timer = null
			},
			randomFood: function () {
				var emptyGrid = []
				for (var i=1; i<this.rowNumber-1; i++) {
				    for (var j=1; j<this.columnNumber-1; j++) {
				        var grid = this.gridList[i][j]
						if (grid.status == this.gridStatus.empty) {
				            emptyGrid.push(grid)
						}
					}
				}
				var randomGrid = emptyGrid[Math.floor(Math.random()*emptyGrid.length)]
				randomGrid.status = this.gridStatus.food
			},
			resetSnake: function () {
				this.snakeGridList = []
				for (var i=4; i>0; i--) {
				    var grid = this.gridList[1][i]
					if (i==4) {
				        grid.status = this.gridStatus.snakeHeader
					} else {
				        grid.status = this.gridStatus.snakeBody
					}
					this.snakeGridList.push(grid)
				}
				this.currentDirection = this.walkDirection.right
			},
			resetGame: function () {
				clearInterval(timer)
				timer = null
				this.gameStatus = 0
				this.speedLevel = 0
				for (var i=0; i<this.rowNumber; i++) {
					var row
					if (i<this.gridList.length) {
						row = this.gridList[i]
						this.gridList.splice(i, 1, row)
					} else  {
						row = []
						this.gridList.push(row)
					}
					for (var j=0; j<this.columnNumber; j++) {
						var grid
						if (j<row.length) {
							grid = row[j]
						} else {
							grid = {}
							row.push(grid)
						}
						grid.row = i
						grid.column = j
						if (((i==0 || i==this.rowNumber-1) && j!=0 && j!=this.columnNumber-1)
							|| ((j==0 || j==this.columnNumber-1) && i!=0 && i!=this.rowNumber-1)) {
						    grid.status = this.gridStatus.wall
						} else {
						    grid.status = this.gridStatus.empty
						}
					}
				}
				this.resetSnake()
				this.startWalk()
				this.randomFood()
			}
		},
		created: function () {
			this.resetGame()
		},
		beforeDestroy: function () {
			this.stopWalk()
		}
	}
	function toast(text, duration) {
		modal.toast({
			message: text,
			duration: duration
		})
	}
</script>

<style scoped>
	.content {
		background-color: white;
		flex-direction: column;
	}
	.footer {
		flex: 1;
		flex-direction: row;
		justify-content: center;
		align-items: center;
	}
	.game-body {
		width: 750px;
		flex-direction: column;
	}
	.game-row {
		flex-direction: row;
	}
	.game-grid {
		flex-direction: row;
		justify-content: center;
		align-items: center;
		padding-left: 2px;
		padding-right: 2px;
		padding-top: 2px;
		padding-bottom: 2px;
	}
	.grid-image {
		background-color: rgba(0,0,0,0);
		position: absolute;
		left: 0;
		top: 0;
		right: 0;
		bottom: 0;
	}
	.reset-button {
		font-size: 30wx;
		color: orange;
	}
	.control-content {
		width: 400px;
		height: 400px;
		flex-direction: column;
	}
	.control-row {
		flex: 1;
		flex-direction: row;
		align-items: center;
	}
	.control-image {
		width: 120px;
		height: 120px;
	}
	.flex-center {
		justify-content: center;
	}
	.flex-space-between {
		justify-content: space-between;
	}
</style>
