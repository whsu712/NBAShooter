# NBAShooter
NBAShooter.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";

contract BaseNBAShooter is Ownable {

    uint256 public constant DAILY_FREE_SHOTS = 5;
    uint256 public constant POINTS_PER_WIN = 120;
    uint256 public constant STARTING_POINTS = 500;

    uint256 public totalShots;

    enum ShotType { TWO_POINT, THREE_POINT, DUNK }

    struct PlayerStats {
        uint256 points;
        uint256 shotsToday;
        uint256 lastShotDay;
        uint256 totalWins;
        uint256 totalShots;
    }

    mapping(address => PlayerStats) public playerStats;

    event ShotResult(address indexed player, ShotType shotType, uint256 pointsEarned, bool success);

    constructor() Ownable(msg.sender) {}

    function _getToday() internal view returns (uint256) {
        return block.timestamp / 1 days;
    }

    function _initPlayer() internal {
        if (playerStats[msg.sender].points == 0) {
            playerStats[msg.sender].points = STARTING_POINTS;
        }
    }

    // Main Game Function - Very low gas
    function takeShot(ShotType shotType) external {
        _initPlayer();
        uint256 today = _getToday();

        if (playerStats[msg.sender].lastShotDay != today) {
            playerStats[msg.sender].shotsToday = 0;
            playerStats[msg.sender].lastShotDay = today;
        }

        require(playerStats[msg.sender].shotsToday < DAILY_FREE_SHOTS, "Daily shots limit reached");

        playerStats[msg.sender].shotsToday++;
        playerStats[msg.sender].totalShots++;
        totalShots++;

        bool success = _calculateHit(shotType);
        uint256 reward = 0;

        if (success) {
            reward = POINTS_PER_WIN;
            playerStats[msg.sender].points += POINTS_PER_WIN;
            playerStats[msg.sender].totalWins++;
        }

        emit ShotResult(msg.sender, shotType, reward, success);
    }

    function _calculateHit(ShotType shotType) internal view returns (bool) {
        uint256 rand = uint256(keccak256(
            abi.encodePacked(
                block.prevrandao,
                block.timestamp,
                msg.sender,
                totalShots
            )
        )) % 100;

        if (shotType == ShotType.TWO_POINT) {
            return rand < 65;   // 65% success rate
        } else if (shotType == ShotType.THREE_POINT) {
            return rand < 38;   // 38% success rate
        } else {
            return rand < 55;   // 55% success rate for Dunk
        }
    }

    // Claim extra points if you run out
    function claimDailyBonus() external {
        _initPlayer();
        uint256 today = _getToday();
        require(playerStats[msg.sender].lastShotDay != today, "Already claimed today");
        
        playerStats[msg.sender].points += 80;
        playerStats[msg.sender].lastShotDay = today;
    }

    // View functions
    function getMyStats() external view returns (
        uint256 points,
        uint256 shotsToday,
        uint256 totalWins,
        uint256 totalShotsTaken
    ) {
        PlayerStats memory stats = playerStats[msg.sender];
        return (stats.points, stats.shotsToday, stats.totalWins, stats.totalShots);
    }

    function getRemainingShots() external view returns (uint256) {
        uint256 today = _getToday();
        if (playerStats[msg.sender].lastShotDay != today) {
            return DAILY_FREE_SHOTS;
        }
        return DAILY_FREE_SHOTS > playerStats[msg.sender].shotsToday ? 
               DAILY_FREE_SHOTS - playerStats[msg.sender].shotsToday : 0;
    }

    // Owner functions
    function addPoints(address player, uint256 amount) external onlyOwner {
        playerStats[player].points += amount;
    }
}

takeShot(0) → 2
takeShot(1) → 3 
takeShot(2) → 
getMyStats() → check score
getRemainingShots() → check times

