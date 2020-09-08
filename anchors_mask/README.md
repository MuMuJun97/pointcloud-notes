
# Anchors mask

生成 `anchors mask` (bool矩阵，[440,500] == grid size)：计算anchors具有的点云个数，并判断是否大于阈值。将大于阈值的选为可用anchors。


# Algorithm

```python


def image_box_region_area(img_cumsum, bbox):
    """check a 2d voxel is contained by a box. used to filter empty
    anchors.
    Summed-area table algorithm:
    ==> W
    ------------------
    |      |         |
    |------A---------B
    |      |         |
    |      |         |
    |----- C---------D
    Iabcd = ID-IB-IC+IA
    Args:
        img_cumsum: [M, H, W](yx) cumsumed image.
        bbox: [N, 4](xyxy) bounding box, 
    """
    N = bbox.shape[0]
    M = img_cumsum.shape[0]
    ret = np.zeros([N, M], dtype=img_cumsum.dtype)
    ID = img_cumsum[:, bbox[:, 3], bbox[:, 2]]
    IA = img_cumsum[:, bbox[:, 1], bbox[:, 0]]
    IB = img_cumsum[:, bbox[:, 3], bbox[:, 0]]
    IC = img_cumsum[:, bbox[:, 1], bbox[:, 2]]
    ret = ID - IB - IC + IA
    return ret


@numba.jit(nopython=True)
def sparse_sum_for_anchors_mask(coors, shape):
    ret = np.zeros(shape, dtype=np.float32)
    for i in range(coors.shape[0]):
        ret[coors[i, 1], coors[i, 2]] += 1
    return ret


@numba.jit(nopython=True)
def fused_get_anchors_area(dense_map, anchors_bv, stride, offset,
                           grid_size):
    anchor_coor = np.zeros(anchors_bv.shape[1:], dtype=np.int32)
    grid_size_x = grid_size[0] - 1
    grid_size_y = grid_size[1] - 1
    N = anchors_bv.shape[0]
    ret = np.zeros((N), dtype=dense_map.dtype)
    for i in range(N):
        anchor_coor[0] = np.floor(
            (anchors_bv[i, 0] - offset[0]) / stride[0])
        anchor_coor[1] = np.floor(
            (anchors_bv[i, 1] - offset[1]) / stride[1])
        anchor_coor[2] = np.floor(
            (anchors_bv[i, 2] - offset[0]) / stride[0])
        anchor_coor[3] = np.floor(
            (anchors_bv[i, 3] - offset[1]) / stride[1])
        anchor_coor[0] = max(anchor_coor[0], 0)
        anchor_coor[1] = max(anchor_coor[1], 0)
        anchor_coor[2] = min(anchor_coor[2], grid_size_x)
        anchor_coor[3] = min(anchor_coor[3], grid_size_y)
        ID = dense_map[anchor_coor[3], anchor_coor[2]]
        IA = dense_map[anchor_coor[1], anchor_coor[0]]
        IB = dense_map[anchor_coor[3], anchor_coor[0]]
        IC = dense_map[anchor_coor[1], anchor_coor[2]]
        ret[i] = ID - IB - IC + IA
    return ret
```
    
## 