// ==UserScript==
// @name         Tagpro Zoom GUI R3
// @version      R3-0.996
// @description  Control Zoom
// @author       Despair
// @include      http://*.koalabeast.com:*
// @include      http://*.jukejuice.com:*
// @include      http://*.newcompte.fr:*
// ==/UserScript==
 
// - Settings - //
 
// - change offset of button positions relative to bottom-right of window
var iconXOffset = 10, iconYOffset = 150;
 
// - start with automatic zoom levels
var startAutoIn = true, startAutoOut = true;
 
// - start centered as spec
var startCentered = true;
 
// - add zoom level and viewport size in top-right of viewport
var showPortData = false, showTileData = true;
 
// - prevent zooming in too much with + and - keys in spec
var boundToLimit = false;
 
// - allow panning the camera when centered
var canPan = true;
 
// - speed(tiles per sec) of camera panning as spec
var panSpeed = 8;
 
// - remap i, j, k, l to arrows to allow panning
var altPanning = true;
 
// - enable all functions as a player - can be considered cheating
var enableZoom = true, showPups = false;
 
/*
 * ##########################
 *    -  end of settings -
 * !! proceed with caution !!
 * ##########################
 */
 
// Variables
var zoomArray = [1, 4/3, 5/3, 2, 2.5, 10/3, 4],//40 30 24 20 16 12 10
    scriptMode = "null", isOn = true, autoZoomIn = false, autoZoomOut = false,
    zoomInLvl = 40, zoomOutLvl = 16, autoZoomInLvl = 40, autoZoomOutLvl = 16,
    worldXCenter = 0, worldYCenter = 0, defaultCenter = [0, 0],
    worldWidth = 0, worldHeight = 0, wasCtrlHeld = false, wasAltHeld = false,
    dataTextOffset = 0, pupList = [], pupGhost = {};
 
// Images - http://imgur.com/a/3o4VM
var img_base = "http://i.imgur.com/NC6G8pH.png",
    img_sOn = "http://i.imgur.com/5blvhrD.png",
    img_sOff = "http://i.imgur.com/W8N6Ia5.png",
    img_zPlus = "http://i.imgur.com/ZMl4njt.png",
    img_zMinus = "http://i.imgur.com/Rm7H6ub.png",
    img_calcOn = "http://i.imgur.com/9Ecly83.png",
    img_calcOff = "http://i.imgur.com/PrcWFF6.png",
    img_zMask = "http://i.imgur.com/0VFdVYw.png",
    img_zAlt = "http://i.imgur.com/EEha17U.png";
 
// INITIALIZATION //
 
// call functions after tagpro loads
tagpro.ready(function(){
   
    // player left while spectating
    tagpro.socket.on("playerLeft", function(id){
        try{
            if(tagpro.spectator) setTimeout(function(){updateZoom();},50);
        }
        catch(error){console.log("Error", error);}
    });
 
    setTimeout(function(){
            startScript();
    },500);
});
 
// call functions with delay
function startScript(){
    findCenter();
    replaceTR();
    zoomDisp();
    makeButtons();
    checkResize();
    onStartCheck();
    setInterval(function(){
        checkResize();
    },500);
}
 
// make buttons
function makeButtons(){
    var a;
    button_base = createButton("img", img_base, 0, 0);//background
    button_active = createButton("img", img_sOn, 16, 16, updateMode);//on/off - color
        button_active.style.background = '#bfbfbf';
    button_zoomPlus = createButton("img", img_zPlus, 2, 21, zoomChange, "+");//zoom out
    button_zoomMinus = createButton("img", img_zMinus, 2, 2, zoomChange, "-");//zoom in
    button_zoomBar = createButton("div", "#ffffff", 48, 4);//zoom bar
        a = button_zoomBar.style; a.height = "5px"; a.width = "8px";
    button_zoomPart = createButton("div", "#ffffff", 48, 4);//zoom bar partial
        a = button_zoomPart.style; a.height = "5px"; a.width = "8px"; a.opacity = 0.5;
    button_zoomMask = createButton("img", img_zMask, 46, 2, resetZoom);//covers zoom bar
    button_calcZIn = createButton("img", img_calcOn, 31, 2, toggleAutoZoom, "in");//calc z in
    button_calcZOut = createButton("img", img_calcOn, 16, 2, toggleAutoZoom, "out");//calc z out
    button_altZoom = createButton("img", img_zAlt, 60, 4);//zoom bar other
}
 
// make a button from input
function createButton(type, input, Xpos, Ypos, action, arg){
    if(!(type == "img" || type == "div")){return;}
    var button = document.createElement(type);
    if(type == "img") button.src = input;
    else if(type == "div") button.style.background = input;
    button.ondragstart = function(){return false;};
    button.style.position = "absolute";
    button.style.right = iconXOffset + Xpos + "px";
    button.style.bottom = iconYOffset + Ypos + "px";
    if(action !== undefined){
        button.onmouseover = function(){button.style.cursor="pointer";};
        button.onclick = function(e){
            wasCtrlHeld = (e.ctrlKey); wasAltHeld = (e.altKey);
            if(arg === undefined){action();}
            else{action(arg);}
        };
    }
    document.body.appendChild(button);
    return button;
}
 
var dispLoc = new PIXI.Point(0, 0);
function zoomDisp(){
    if(showPortData || showTileData){
        if(showPortData) dataTextOffset += 70;
        if(showTileData) dataTextOffset += 50;
        tagpro.ui.sprites.zoomDisplay = tagpro.renderer.prettyText("=3");
        tagpro.renderer.layers.ui.addChild(tagpro.ui.sprites.zoomDisplay);
        dispLoc.set(tagpro.renderer.canvas.width - dataTextOffset, 10);
        tagpro.ui.sprites.zoomDisplay.position = dispLoc;
        setInterval(function(){
            updateZoomDisp();
        }, 100);
    }
}
 
// find world midpoint from wall tiles
function findCenter(){
    var i = 0, j = 0;
    var mapWidth = tagpro.map.length, mapHeight = tagpro.map[0].length;
    var earlyRow = 999, lateRow = -1, earlyCol = 999, lateCol = -1;
    for (i = 0; i < mapWidth; i++){
        var column = tagpro.map[i].slice(0);//get copy not object reference
        var firstWall = 0, lastWall = 0;
        for (j = 0; j < mapHeight; j++){
            column[j] = Math.floor(column[j]);
            if(column[j] == 6){
                pupList.push({x:i, y:j});
                if(!pupGhost[i]) pupGhost[i] = {};
                pupGhost[i][j] = tagpro.tiles.draw(tagpro.renderer.layers.midground, 6, {x:i*40, y:j*40});
                pupGhost[i][j].visible = false;
            }
        }
        firstWall = column.indexOf(1);
        if(firstWall < earlyRow && firstWall != -1) earlyRow = firstWall;
        lastWall = column.lastIndexOf(1);
        if(lastWall > lateRow) lateRow = lastWall;
        if(firstWall != -1){
            if(i < earlyCol) earlyCol = i;
            if(i > lateCol) lateCol = i;
        }
    }
    var hasWalls = (earlyRow != 999);
    worldWidth = hasWalls ? 1 + lateCol - earlyCol : mapWidth;
    worldHeight = hasWalls ? 1 + lateRow - earlyRow : mapHeight;
    worldXCenter = hasWalls ? earlyCol + (worldWidth / 2) : mapWidth / 2;
    worldYCenter = hasWalls ? earlyRow + (worldHeight / 2) : mapHeight / 2;
    defaultCenter = [worldXCenter, worldYCenter];
}
 
// set values after startup
function onStartCheck(){
    var trodvs = tagpro.renderer.options.disableViewportScaling;
    if(!trodvs) trodvs = true;
    if(tagpro.spectator){
        scriptMode = startCentered ? "center" : "follow";
        specLoop = setInterval(function(){
            checkSpec();
            if(!tagpro.spectator) clearInterval(specLoop);
        }, 1000);
    }
    else scriptMode = "self";
    if(startAutoIn){
        autoZoomIn = true;
        //zoomInLvl = autoZoomInLvl;
    }
    if(startAutoOut){
        autoZoomOut = true;
        //zoomOutLvl = autoZoomOutLvl;
    }
    updateMainMask();
    updateAutoMask();
    updateZoom();
}
 
// KEY CONTROL //
 
$(document).keydown(function(key){keyUpdate(key, "down");});
$(document).keyup(function(key){keyUpdate(key, "up");});
 
var arrows = [0,0,0,0,false,false];
function keyUpdate(e, state){
    e = e || window.event;
    var charCode = e.which || e.keyCode,
        useAltPan = false;
    if(document.getElementById("chat").style.display !== "inline-block"){
        // - camera panning - //
        if(altPanning && canPan){
            if(charCode>=73 && charCode<=76){
                var keymap = {73:38, 74:37, 75:40, 76:39};
                charCode = keymap[charCode];
                useAltPan = true;
            }
        }
        if((tagpro.spectator || useAltPan) && (!tagpro.viewport.followPlayer && canPan)){
            arrows[4] = e.shiftKey;
            arrows[5] = e.ctrlKey || e.altKey;
            if(state == "down"){
                if(charCode>=37 && charCode<=40){
                    arrows[charCode-37] = 1;
                }
            }else if(state == "up"){
                if(charCode>=37 && charCode<=40){
                    arrows[charCode-37] = 0;
                }
            }
        }
        //console.log(charCode +" "+state);
        /*
        if(state == "down"){//triggered continuously while key is held
            //
        }
        */
        if(state == "up"){
            if(charCode == 90) updateZoom();
            if(charCode == 86){
                worldXCenter = defaultCenter[0];
                worldYCenter = defaultCenter[1];
            }
            if(tagpro.spectator){
                if(charCode == 67){
                    if(scriptMode == "flag") forceMode("center");
                    else syncMode();
                }
                switch(charCode){
                    case 107:
                    case 109:
                        syncZoom();
                        break;
                    case 81:
                    case 87:
                    case 65:
                    case 83:
                        forceMode("follow");
                        break;
                    case 68:
                        forceMode("flag");
                        break;
                    default:
                        break;
                }
            }
        }
    }
}
 
if(canPan){
    setInterval(function(){
        updateCenter();
    }, 50);
}
function updateCenter(){
    var speed = panSpeed/20;
    if(arrows[4]) speed *= 2;
    if(arrows[5]) speed /= 2;
    worldXCenter += speed * (arrows[2]-arrows[0]);
    worldYCenter += speed * (arrows[3]-arrows[1]);
}
 
function syncZoom(){
    if(tagpro.spectator){
        var a = Math.round(40/tagpro.zoom);
        if(a === 0){a = 1; alert("no u pliss");}
        if(tagpro.viewport.followPlayer && !autoZoomIn) zoomInLvl = a;
        else if(!tagpro.viewport.followPlayer && !autoZoomOut) zoomOutLvl = a;
        tagpro.zoom = 40/a;
        updateZoom();
    }
}
 
function syncMode(){
    setTimeout(function(){
        if(tagpro.viewport.followPlayer){
            if(scriptMode != "follow"){
                scriptMode = "follow";
                updateMainMask();
                updateZoom();
            }
        }
        else{
            if(scriptMode != "center"){
                scriptMode = "center";
                updateMainMask();
                updateZoom();
            }
        }
    },50);
}
 
function forceMode(input){
    setTimeout(function(){
        scriptMode = input;
        updateMainMask();
        updateZoom();
    },50);
}
 
// ZOOM CONTROLLER //
 
// change zoom level and focus
function updateZoom(){
    if(isOn){
        var mode = (scriptMode != "flag") ? scriptMode : FC.mode;
        if(!tagpro.spectator && !enableZoom){
            zoomOutLvl = Math.max(zoomOutLvl, autoZoomOutLvl);
            zoomInLvl = Math.max(zoomInLvl, autoZoomInLvl);
            mode = "self";
        }
        if(mode == "center"){
            tagpro.viewport.followPlayer = false;
            tagpro.zoom = 40 / ((autoZoomOut) ? autoZoomOutLvl : zoomOutLvl);
        }
        else if(mode == "self" || mode == "follow"){
            tagpro.viewport.followPlayer = true;
            tagpro.zoom = 40 / ((autoZoomIn) ? autoZoomInLvl : zoomInLvl);
            if(scriptMode == "flag"){
                var x = Number((FC.red !== 0) ? FC.red : FC.blue);
                tagpro.socket.emit((tagpro.players[x].team === 1) ? "redflagcarrier" : "blueflagcarrier");
            }
        }
    }
    updateZoomBar();
}
 
// - FLAG MODE - //
 
setInterval(function(){followFlags();}, 100);
 
var FC = {red:0, blue:0, state:"null", mode:"null"};
function followFlags(){
    if(tagpro.spectator){
        if(isOn && scriptMode == "flag"){
            checkFC("both");
            //console.log("red-"+FC.red+" blue-"+FC.blue);
            if((FC.red !== 0 && FC.blue !== 0) || (FC.red === 0 && FC.blue === 0)){
                if(FC.state != "center"){
                    FC.state = "center";
                    changeMode("center");
                }
            }else if(FC.red !== 0 && FC.blue === 0){
                if(FC.state != "followRed"){
                    FC.state = "followRed";
                    changeMode("followRed");
                }
            }else if(FC.red === 0 && FC.blue !== 0){
                if(FC.state != "followBlue"){
                    FC.state = "followBlue";
                    changeMode("followBlue");
                }
            }
        }
    }
}
 
var modeCooldown;
function changeMode(input){
    var redHasYellow = tagpro.players[FC.red] ? tagpro.players[FC.red].flag == 3 : false,
        blueHasYellow = tagpro.players[FC.blue] ? tagpro.players[FC.blue].flag == 3 : false,
        isInstant = redHasYellow || blueHasYellow;
    clearTimeout(modeCooldown);
    modeCooldown = setTimeout(function(){
            var x = ((FC.red !== 0 && FC.blue !== 0) || (FC.red === 0 && FC.blue === 0));
        if(input == 'center' && x){
            FC.mode = 'center';
            updateZoom();
        }else if((input == 'followRed' || input == 'followBlue') && !x){
            FC.mode = 'follow';
            updateZoom();
        }
    },!isInstant ? 1000 : 50);
}
 
function checkFC(input){
    if(input == "both"){
        checkFC("red");
        checkFC("blue");
    }else{
        if(tagpro.players[FC[input]]){
            if(tagpro.players[FC[input]].flag === null){
                FC[input] = 0;
                findFC();
            }
        }else{
            FC[input] = 0;
            findFC();
        }
    }
}
 
function findFC(){
    for(var key in tagpro.players){
        if(tagpro.players.hasOwnProperty(key)){
            var x = tagpro.players[key];
            if(x.flag !== null){
                if(x.team == 1) FC.red = key;
                else if(x.team == 2) FC.blue = key;
            }
        }
    }
}
 
// ICON UPDATE //
 
// change camera color based off of mode
function updateMainMask(){
    var color = '#333333';
    if(scriptMode == "center") color = '#e5e500';
    else if(scriptMode == "self") color = '#00e500';
    else if(scriptMode == "follow") color = '#00e5e5';
    else if(scriptMode == "flag"){
        color = 'linear-gradient(135deg, #ff6565, #ff6565 25%, #6f91ff 75%, #6f91ff)';
    }
    button_active.style.background = color;
    button_active.src = (isOn) ? img_sOn : img_sOff;
}
 
// show active state of calculated zoom
function updateAutoMask(){
    button_calcZIn.src = (autoZoomIn) ? img_calcOn : img_calcOff;
    button_calcZOut.src = (autoZoomOut) ? img_calcOn : img_calcOff;
}
 
// update zoom bar
function updateZoomBar(){
    var mainDisplay = 0, sideDisplay = 0, mainOffset = 0, sideOffset = 0;
    if(scriptMode == "self" || scriptMode == "follow"){
        mainDisplay = zoomInLvl;
        sideDisplay = zoomOutLvl;
    }else{
        mainDisplay = zoomOutLvl;
        sideDisplay = zoomInLvl;
    }
    mainOffset = calcZoomBarOffset(mainDisplay);
    button_zoomBar.style.height = 5 + mainOffset - (mainOffset % 4) + "px";
    button_zoomPart.style.bottom = iconYOffset + 6 + mainOffset + "px";
    button_zoomPart.style.opacity = (mainOffset % 4 !== 0) ? 0.5 : 0;
    sideOffset = calcZoomBarOffset(sideDisplay);
    button_altZoom.style.bottom = iconYOffset + 4 + sideOffset + "px";
}
 
// get offset for zoom indicators
function calcZoomBarOffset(input){
    var i, output = 0, loop = true;
    if(input < 40){
        for(i = 0; loop; i++){
            if(i >= 6){
                output = 26; loop = false;
            }
            if(input == 40/zoomArray[i]){
                output = i*4; loop = false;
            }
            else if(40/zoomArray[i] > input && input > 40/zoomArray[i+1]){
                output = (i*4)+2; loop = false;
            }
        }
    }
    return output;
}
 
// INTERACTIVE BUTTONS //
 
// change script state
function updateMode(){
    if(!wasCtrlHeld){
        if(tagpro.spectator){
            if(scriptMode == "center") scriptMode = (wasAltHeld) ? "flag" : "follow";
            else scriptMode = (!wasAltHeld) ? "center" : ((scriptMode != "flag") ? "flag" : "follow" );
        }else{
            if(scriptMode == "self"){scriptMode = "center";}
            else{scriptMode = "self";}
        }
    }else{isOn = !isOn;}
    updateMainMask();
    updateZoom();
}
 
// toggle auto zoom calculation
function toggleAutoZoom(input){
    if(input == "in") autoZoomIn = !autoZoomIn;
    else if(input == "out") autoZoomOut = !autoZoomOut;
    updateAutoMask();
    updateZoom();
}
 
// increase or decrease zoom level
function zoomChange(input){
    var changeZoomIn = false, changeValue = 0;
    if(!wasAltHeld) changeZoomIn = false;
    else changeZoomIn = true;
    if(scriptMode == "self" || scriptMode == "follow") changeZoomIn = !changeZoomIn;
    if(!wasCtrlHeld){
        var curZoom = 0, i, loop = true, next, prev;
        if(!changeZoomIn) curZoom = zoomOutLvl;
        else curZoom = zoomInLvl;
        for(i = 0; loop; i++){
            if(i >= 6) loop = false;
            if(curZoom == 40/zoomArray[i]){
                next = zoomArray[i+1]; prev = zoomArray[i-1]; loop = false;
            }else if(40/zoomArray[i] > curZoom && curZoom > 40/zoomArray[i+1]){
                next = zoomArray[i+1]; prev = zoomArray[i]; loop = false;
            }
        }
        if(prev === undefined) prev = zoomArray[6];
        if(next === undefined) next = zoomArray[0];
        if(input == "+") changeValue = curZoom - (40/next);
        else changeValue = (40/prev) - curZoom;
    }
    else changeValue = 1;
    if(input == "+") changeValue *= -1;
    var zoomValue = (changeZoomIn) ? zoomInLvl : zoomOutLvl;
    zoomValue += changeValue;
    zoomValue = Math.max(Math.min(zoomValue, 40), 1);
    if(changeZoomIn) zoomInLvl = zoomValue;
    else zoomOutLvl = zoomValue;
    updateZoom();
}
 
// reset zoom
function resetZoom(){
    var changeZoomIn = false;
    if(!wasAltHeld) changeZoomIn = false;
    else changeZoomIn = true;
    if(scriptMode == "self" || scriptMode == "follow") changeZoomIn = !changeZoomIn;
    if(!changeZoomIn){
        if(zoomOutLvl != 40/zoomArray[0]) zoomOutLvl = 40/zoomArray[0];
        else zoomOutLvl = 40/zoomArray[4];
    }else{
        if(zoomInLvl != 40/zoomArray[0]) zoomInLvl = 40/zoomArray[0];
        else zoomInLvl = 40/zoomArray[4];
    }
    updateZoom();
}
 
// AUTO UPDATE //
 
// resize detector
var vpHighCache = 0;
function checkResize(){
    var vpHighCheck = tagpro.renderer.canvas.height;
    if(vpHighCheck != vpHighCache){
        vpHighCache = vpHighCheck;
        calcAutoZoomIn();
        calcAutoZoomOut();
        updateZoom();
        updatePositions();
    }
}
 
setInterval(function(){updatePupSight();}, 100);
function updatePupSight(){
    for(var i=0; i < pupList.length; i++){
        var canSee = false, seeX, seeY;
        if(showPups || tagpro.spectator) canSee = true;
        else{
            seeX = Math.abs((pupList[i].x*40) - tagpro.players[tagpro.playerId].x) < 660;
            seeY = Math.abs((pupList[i].y*40) - tagpro.players[tagpro.playerId].y) < 420;
            canSee = (seeX && seeY);
        }
        tagpro.renderer.dynamicSprites[pupList[i].x][pupList[i].y].visible = canSee ? true : false;
        pupGhost[pupList[i].x][pupList[i].y].visible = canSee ? false : true;
    }
}
 
function updatePositions(){
    if(showTileData || showPortData){
        dispLoc.set(tagpro.renderer.canvas.width - dataTextOffset, 10);
        tagpro.ui.sprites.zoomDisplay.position = dispLoc;
    }
}
 
// show zoom level
function updateZoomDisp(){
    var x = "";
    if(showPortData) x += tagpro.renderer.canvas.width + " X " + tagpro.renderer.canvas.height;
    if(showPortData && showTileData) x += " - ";
    if(showTileData) x += Math.round(400/tagpro.zoom) / 10 + " px";
    tagpro.ui.sprites.zoomDisplay.setText(x);
}
 
// join from spectator detector
function checkSpec(){
    if(!tagpro.spectator){
        scriptMode = "self";
        updateMainMask();
        updateZoom();
    }
}
 
// CALCULATE ZOOM LEVELS //
 
// calculate auto zoomin value
function calcAutoZoomIn(){
    var vpHR = tagpro.renderer.canvas.height / 800;
    autoZoomInLvl = Math.max(Math.round(40 * vpHR), 1);
}
 
// calculate auto zoomout value
function calcAutoZoomOut(){
    var wideRatio = ((worldWidth * 40) - 60) / tagpro.renderer.canvas.width,
        highRatio = ((worldHeight * 40) - 60) / tagpro.renderer.canvas.height,
        zoomForWidth = Math.floor(40 / wideRatio),
        zoomForheight = Math.floor(40 / highRatio);
    autoZoomOutLvl = Math.max(Math.min(zoomForWidth, zoomForheight, 40), 1);
}
 
// REPLACE TAGPRO FUNCTIONS //
function replaceTR(){
   
    var tr = tagpro.renderer;
   
    tr.centerCameraPosition = function (){
        tr.centerContainerToPoint(worldXCenter*tagpro.TILE_SIZE, worldYCenter*tagpro.TILE_SIZE);
    };
   
    tr.updateCameraZoom = function(){
        var resizeScaleFactor = tr.options.disableViewportScaling ? 1 : this.vpHeight / tr.canvas_height;
        tagpro.zoom = tagpro.zoom + tagpro.zooming > 0.1 ? tagpro.zoom + tagpro.zooming : tagpro.zoom;
        if(boundToLimit) tagpro.zoom = Math.max(tagpro.zoom, 1);
        var scale = new PIXI.Point(1.0 / tagpro.zoom * resizeScaleFactor, 1.0 / tagpro.zoom * resizeScaleFactor);
        var inverseScale = new PIXI.Point(tagpro.zoom, tagpro.zoom);
        tr.gameContainer.scale = scale;
        for(var player in tagpro.players){
            if(!tagpro.players.hasOwnProperty(player)) continue;
            var p = tagpro.players[player];
            var info = p.sprites.info;
            info.x = 19 - (tagpro.zoom * 19);
            info.scale = inverseScale;
            if(p.sprites){
                var visibility = tagpro.zoom < tr.zoomThreshold;
                if(p.sprites.flair) p.sprites.flair.visible = visibility;
                if(p.sprites.degrees) p.sprites.degrees.visible = visibility;
            }
        }
    };
}
