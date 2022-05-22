import { Buffer } from 'buffer';

export class TDMParser {

    // calc difference between NI & UNIX timestamp
    static t_NI = new Date("1904-01-01 00:00:00");
    static t_UNIX = new Date("1970-01-01 00:00:00");
    static t_dif = (this.t_UNIX.getTime() - this.t_NI.getTime()) / 1000;

    // absolute poniter to read the bytes in the buffer
    static pntr = 0

    static tocProperties: any = {
        'kTocMetaData': (1 << 1),
        'kTocRawData': (1 << 3),
        'kTocDAQmxRawData': (1 << 7),
        'kTocInterleavedData': (1 << 5),
        'kTocBigEndian': (1 << 6),
        'kTocNewObjList': (1 << 2),
    }

    static tdsDataTypesDefined: any = { // might be incorrect !!!!
        "00000000": 'tdsTypeVoid',
        "01000000": 'tdsTypeI8',
        "02000000": 'tdsTypeI16',
        "03000000": 'tdsTypeI32',
        "04000000": 'tdsTypeI64',
        "05000000": 'tdsTypeU8',
        "06000000": 'tdsTypeU16',
        "07000000": 'tdsTypeU32',
        "08000000": 'tdsTypeU64',
        "09000000": 'tdsTypeSingleFloat',
        "0A000000": 'tdsTypeDoubleFloat',
        "0B000000": 'tdsTypeExtendedFloat',
        "1A000000": 'tdsTypeDoubleFloatWithUnit',
        "1B000000": 'tdsTypeExtendedFloatWithUnit',
        "19000000": 'tdsTypeSingleFloatWithUnit',
        "20000000": 'tdsTypeString',
        "21000000": 'tdsTypeBoolean',
        "44000000": 'tdsTypeTimeStamp',
        "4F000000": 'tdsTypeFixedPoint',
        "08000c00": 'tdsTypeComplexSingleFloat',
        "10000d00": 'tdsTypeComplexDoubleFloat',
        "FFFFFFFF": 'tdsTypeDAQmxRawData',
    }

    static dataTypeLength: any = (datatype: any) => {
        if (['tdsTypeVoid'].includes(datatype)) return 0
        if (['tdsTypeI8', 'tdsTypeU8', 'tdsTypeBoolean'].includes(datatype)) return 1
        if (['tdsTypeI16', 'tdsTypeU16'].includes(datatype)) return 2
        if (['tdsTypeI32', 'tdsTypeU32', 'tdsTypeSingleFloat', 'tdsTypeSingleFloatWithUnit'].includes(datatype)) return 4
        if (['tdsTypeI64', 'tdsTypeU64', 'tdsTypeDoubleFloat', 'tdsTypeDoubleFloatWithUnit'].includes(datatype)) return 8
        if (['tdsTypeTimeStamp'].includes(datatype)) return 16
        if ([
            'tdsTypeString',
            'tdsTypeExtendedFloat',
            'tdsTypeExtendedFloatWithUnit',
            'tdsTypeDAQmxRawData',
        ].includes(datatype)) return false
    }

    static readFileDataAsBase64 = (f: any, group: string) => {

        return new Promise((resolve, reject) => {
            const reader = new FileReader();

            let buffer: any = []    // storage of file content
            this.pntr = 0           // reset buffer pointer
            const file: any = {}     // create result variable

            reader.onload = (event: any) => {

                // fill buffer
                (new Uint8Array(event.target.result)).forEach((element: any) => {
                    buffer.push(Number(element).toString(16).padStart(2, '0').toUpperCase())
                })

                file.segment = this.readLeadIn(buffer)   // read lead in

                if (file.segment.kTocMetaData) {           // read metadata
                    file.metadata = this.readMetadata(
                        buffer,
                        file.segment.lenMetadata,
                        file.segment["kTocBigEndian"]
                    )
                }

                file.group = this.readGroup(buffer)      // read group

                file.group.channels = this.readChannels( // read channels
                    buffer,
                    file.group,
                    file.segment.objects - 2,
                    file.segment["kTocBigEndian"],
                    group
                )

                this.readRawData(                        // read rawData
                    buffer,
                    file.segment.rawDataOffset,
                    file.segment.nextSegment,
                    file.segment["kTocInterleavedData"],
                    file.segment.objects - 3,
                    file.group
                )

                // read next segment
                // if ( file.segment.nextSegment != buffer.length  ){
                //     let seg2 = this.read(buffer, this.pntr = file.segment.nextSegment, 13, true)
                //     console.log(file.segment.nextSegment, buffer.length, seg2)
                // }

                console.log(file)

                resolve(file);
            };

            reader.onerror = (err) => {
                reject(err);
            };

            reader.readAsArrayBuffer(f);

        });
    }

    static read = (buffer: any, start: number, N: number, type?: boolean) => {
        var str = '';

        for (var n = start; n < start + N; n++) {
            if (type) str += buffer[n]
            else str += String.fromCharCode(parseInt(buffer[n], 16)).replace("Â", "");
        }

        return str
    }

    static readNumber = (buffer: any, start: number, N: number, MSB?: boolean) => {

        var str = '';

        if (MSB) {
            for (var n = start; n < start + N; n++) {
                str += buffer[n]
            }
        }
        else {
            for (var n = start + N - 1; n >= start; n--) {
                str += buffer[n]
            }
        }

        return parseInt(str, 16)
    }

    static readLeadIn = (buffer: any) => {
        let segment_object: any = {}

        if (this.read(buffer, this.pntr, 4) === "TDSm") {

            this.pntr += 4

            Object.keys(this.tocProperties).forEach((toc: any) => {
                let toc_enabled = (this.readNumber(buffer, this.pntr, 4) & this.tocProperties[toc]) != 0
                segment_object[toc] = toc_enabled
            })

            segment_object.version = this.readNumber(buffer, this.pntr += 4, 4)
            segment_object.nextSegment = this.readNumber(buffer, this.pntr += 4, 8) + this.pntr + 16
            segment_object.rawDataOffset = this.readNumber(buffer, this.pntr += 8, 8) + this.pntr + 16
            segment_object.objects = this.readNumber(buffer, this.pntr += 8, 4) // number of objects
            let len = this.readNumber(buffer, this.pntr += 4, 4) // length of segment name
            segment_object.name = this.read(buffer, this.pntr += 4, len) // segment name
            segment_object.rawDataIndex = this.read(buffer, this.pntr += len, 4, true)
            segment_object.lenMetadata = this.readNumber(buffer, this.pntr += 4, 4)

            this.pntr += 4

            return segment_object
        }

        return 0

    }

    static readMetadata = (buffer: any, lenMetadata: number, endianness: boolean) => {
        let metadata = []

        for (let i = 0; i < lenMetadata; i++) {
            metadata.push(this.readProperty(buffer, endianness))
        }

        return metadata
    }

    static readProperty = (buffer: any, endianness: any) => {
        let property: any = {}

        let len_name = this.readNumber(buffer, this.pntr, 4)
        property.name = this.read(buffer, this.pntr += 4, len_name)
        property.type = this.tdsDataTypesDefined[this.read(buffer, this.pntr += len_name, 4, true)]

        this.pntr += 4

        switch (property.type) {
            case "tdsTypeString":
                let len_val = this.readNumber(buffer, this.pntr, 4)
                property.value = this.read(buffer, this.pntr += 4, len_val)
                this.pntr += len_val
                break;
            case "tdsTypeTimeStamp":
                property.value = this.readTimestamp(buffer, this.dataTypeLength(property.type) / 2)
                this.pntr += this.dataTypeLength(property.type)
                break;
            case "tdsTypeDoubleFloat":
                property.value = this.hexStringToFloat(
                    this.read(buffer, this.pntr, this.dataTypeLength(property.type), true),
                    endianness
                )
                this.pntr += this.dataTypeLength(property.type)
                break;
            default:
                this.dataTypeLength(property.type)
                property.value = this.readNumber(buffer, this.pntr, this.dataTypeLength(property.type))
                this.pntr += this.dataTypeLength(property.type)
                break;
        }

        return property
    }

    static readGroup = (buffer: any) => {
        let group: any = {}

        let len = this.readNumber(buffer, this.pntr, 4)
        group.name = this.read(buffer, this.pntr += 4, len)
        group.rawDataIndex = this.read(buffer, this.pntr += len, 4, true)
        group.numberPropsChanged = this.read(buffer, this.pntr += 4, 4, true)

        this.pntr += 4

        return group
    }

    static readChannels = (buffer: any, group: any, len: number, endianness: boolean, custom_group: string) => {
        let channels: any = []

        for (let i = 0; i < len; i++) {
            let channel = this.readChannel(buffer)
            channel.yAxisID = 'y'
            channel.group = custom_group || ""
            channel.data = []
            channel.properties = []

            for (let j = 0; j < channel.numProps; j++) {
                channel.properties.push(this.readProperty(buffer, endianness))
            }

            // no need
            this.prepareChannelLabel(group, channel)

            channels.push(channel)
        }

        return channels
    }

    static readChannel = (buffer: any) => {
        let channel: any = {}

        let lenN = this.readNumber(buffer, this.pntr, 4)
        channel.label = this.read(buffer, this.pntr += 4, lenN)
        this.read(buffer, this.pntr += lenN, 4, true)
        channel.type = this.tdsDataTypesDefined[this.read(buffer, this.pntr += 4, 4, true)] // type rawData
        this.read(buffer, this.pntr += 4, 4, true) // dimension of array (must be 1)
        this.read(buffer, this.pntr += 4, 8, true) // length rawData
        channel.numProps = this.readNumber(buffer, this.pntr += 8, 4)

        this.pntr += 4

        return channel
    }

    static readRawData = (buffer: any, rawDataOffset: number, nextSegment: number, interleaved: boolean, len_channels: number, group: any) => {

        var timestamp = 0
        var index = 0

        this.pntr = rawDataOffset

        try {

            while (this.pntr < nextSegment) {


                if ("Timestamp []" == group.channels[0].label) { // read timestamp 
                    timestamp = this.readTimestamp2(buffer, 8)
                    group.channels[0].data.push(timestamp)
                }

                let offset = this.dataTypeLength("tdsTypeDoubleFloat") // TODO: the metadata of each channel must be read

                if (interleaved) { // read interleaved rawdata
                    for (let k = 0; k < len_channels; k++) {

                        let e = this.read(buffer, this.pntr += offset, offset, true)

                        group.channels[k + 1].data.push({
                            x: timestamp,
                            y: this.hexStringToFloat(e, false)
                        })

                    }
                }
                else {
                    // TODO: read not interleaved rawdata
                    let e = this.read(buffer, this.pntr += offset, offset, true)

                    group.channels[0].data.push({
                        x: index++,
                        y: this.hexStringToFloat(e, false)
                    })

                }

                this.pntr += this.dataTypeLength("tdsTypeTimeStamp")
            }

        } catch (error) { console.log(error) }
    }

    static readTimestamp = (buffer: any, len: number) => { // read timestamp in metadata
        return this.readNumber(buffer, this.pntr + 8, len) + this.readNumber(buffer, this.pntr, len, true) * 2 ** (-64) - this.t_dif
    }

    static readTimestamp2 = (buffer: any, len: number) => { // read timestamp in rawdata
        return parseInt(
            String(this.readNumber(buffer, this.pntr, len) - this.t_dif) +
            (this.readNumber(buffer, this.pntr + 8, len) * 2 ** (-64)).toFixed(3).replace("0.", "")
        )
    }

    static hexStringToFloat(hexString: string, endianness: boolean): any { // get value of 8 Bytes
        if (endianness) return Buffer.from(hexString, 'hex').readDoubleBE(0);
        else return Buffer.from(hexString, 'hex').readDoubleLE(0);
    }

    static prepareChannelLabel = (group: any, channel: any) => { // no need
        channel.label = channel.label.replace(group.name + "/'", "").replace("'", "").replace(" ", "")
        if (channel.properties.length > 0) channel.label += " [" + channel.properties[0].value + "]"
        else channel.label += " []"
    }

}
