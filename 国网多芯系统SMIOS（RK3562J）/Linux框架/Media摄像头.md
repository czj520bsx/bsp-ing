
sh ./insmod_media.sh
./media_link_setup
sh ./grab_one.sh /root/photo.png



./media_pad_fmt --set
./media_pad_fmt -d /dev/media1 | grep -A2 'rkisp-isp'
./media_link_setup
./v4l2_grab -d /dev/video11 -o /root/photo.nv12 -f NV12 -w 2592 -h 1944 -n 5


MEDIA+CIF+ISP+sensor


echo 5-0037 > /sys/bus/i2c/drivers/gc05a2/unbind



GC05A2 (i2c5@0x37)
  → csi2-dphy0
  → mipi0-csi2
  → rkcif-mipi-lvds (stream id0)
  → rkcif-mipi-lvds-sditf
  → rkisp-vir0
  → rkisp_mainpath (/dev/videoX，常见即 video11)